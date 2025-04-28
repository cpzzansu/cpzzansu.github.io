---
title: React 클라우드 스토리지 대용량 파일 업로드 취소 기능 구현
description: >-
  대용량 파일을 업로드 할 때, "취소" 할 수 있도록 구현하기
author: author_js
date: 2025-04-28 12:01:00 +0900
categories: [React, Cloud Storage]
tags: [html, react, javascript, css, file, upload, modal, redux]
pin: true
---

대용량 파일을 업로드할 때, 전송 도중 "취소"할 수 있도록 구현하기, Axios의 AbortController와 전역 취소 서비스를 결합해, 
페이지 어떤 컴포넌트에서든 쉽게 "업로드 취소" 할 수 있도록 구현하는 기능을 살펴봅시다.

## 1. 업로드 API 수정하기

기존에 단순히 onUploadProgress 콜백만을 써서 파일의 전송의 진행상황을 전송해주던 uploadFileApi를 AbortController를 이용해 
"업로드 취소"함수를 함께 반환하도록 바꿉니다.

```javascript
// services/uploadService.js
import axios from 'axios';

export function uploadFileApi({ formData, onProgress, folderId }) {
  // ① AbortController 생성
  const controller = new AbortController();

  // ② Axios 요청에 signal 전달
  const promise = axios.post(
    `/api/drive/upload?folderId=${folderId}`,
    formData,
    {
      headers: { 'Content-Type': 'multipart/form-data' },
      signal: controller.signal,
      onUploadProgress: (e) => {
        if (!e.total) return;
        const percent = Math.round((e.loaded * 100) / e.total);
        onProgress(percent);
      },
    }
  ).then(res => res.data);

  // ③ promise 와 cancel 함수를 반환
  return {
    promise,
    cancel: () => controller.abort(),
  };
}
```

- axios의 signal 옵션에 controller.signal 을 넘기게 되면, controller.abort() 호출 시 네트워크 요청이 즉시 중단됩니다.

## 2. 전역 취소 서비스 만들기

함수를 Redux 스토어에 넣으면 직렬화 오류가 발생합니다. 그래서 모듈 싱클톤으로 취소 함수만 관리합니다.

```javascript
// services/uploadCancelService.js
const cancelMap = {};

export const registerCancel = (id, cancelFn) => {
  cancelMap[id] = cancelFn;
};

export const cancelUpload = (id) => {
  if (cancelMap[id]) {
    cancelMap[id]();      // 업로드 중단
    delete cancelMap[id];
  }
};

export const cancelAllUploads = () => {
  Object.values(cancelMap).forEach(fn => fn());
  Object.keys(cancelMap).forEach(k => delete cancelMap[k]);
};
```

- registerCancel(id, fn) 으로 각 업로드의 취소 함수를 ID별로 저장
- cancelUpload(id) 혹은 cancelAllUploads() 로 언제든 취소할 수 있습니다.

## 3. 업로드 매니저 컴포넌트

파일 선택부터 API 호출, 취소 함수 등록까지 처리하는 UploadManager 컴포넌트의 예시입니다.

```javascript
// components/UploadManager.jsx
import React, { useRef } from 'react';
import { uploadFileApi } from '../services/uploadService';
import { registerCancel } from '../services/uploadCancelService';

export default function UploadManager({ folderId, onAddMeta }) {
  const fileInputRef = useRef();

  const handleFiles = (e) => {
    const files = Array.from(e.target.files);

    files.forEach(file => {
      const id = `${file.name}-${file.size}-${Date.now()}`;

      // 1) 메타데이터(이름·크기 등) 상위 컴포넌트에 알림
      onAddMeta({ id, name: file.name, size: file.size, progress: 0 });

      // 2) API 호출 & cancel 등록
      const formData = new FormData();
      formData.append('file', file);

      const { promise, cancel } = uploadFileApi({
        formData,
        folderId,
        onProgress: (pct) => onAddMeta({ id, progress: pct }),
      });

      registerCancel(id, cancel);

      // 3) 완료 시 원하는 후처리
      promise.then(() => {
        // 예: 리스트 리패치
      }).catch(err => {
        if (err.name === 'CanceledError') {
          console.log(`업로드(${id})가 취소되었습니다`);
        }
      });
    });
  };

  return (
    <>
      <input
        type="file"
        multiple
        style={{ display: 'none' }}
        ref={fileInputRef}
        onChange={handleFiles}
      />
      <button onClick={() => fileInputRef.current.click()}>
        파일 업로드
      </button>
    </>
  );
}
```

- onAddMeta 콜백 함수로 상위 컴포넌트에 메타데이터만 전송
- uploadFileApi 가 제공한 cancel 함수를 ID별로 uploadCancelService에 등록합니다.

## 4. 업로드 모달 컴포넌트

```javascript
// components/UploadModal.jsx
import React from 'react';
import { useSelector } from 'react-redux';
import { cancelUpload, cancelAllUploads } from '../services/uploadCancelService';

export default function UploadModal() {
  const uploads = useSelector(state => state.upload.uploadFiles);

  return (
    <div className="upload-modal">
      <div className="modal-header">
        <h3>업로드 중</h3>
        <button onClick={cancelAllUploads}>전체 취소</button>
      </div>
      <div className="modal-body">
        {uploads.map(f => (
          <div key={f.id} className="upload-item">
            <span className="file-name">{f.name}</span>
            <progress value={f.progress} max={100} />
            <button onClick={() => cancelUpload(f.id)}>취소</button>
          </div>
        ))}
      </div>
    </div>
  );
}
```

- cancelUpload(f.id) 로 업로드 파일 개별로 취소
- cancelAllUploads() 로 전체 취소가 가능합니다.

## 5. 전체 구조 요약

```text
App
 └─ UploadManager            // 파일 선택 + API 호출 + cancel 등록
 └─ UploadModal              // 전역 업로드 현황 + 취소 버튼
```

1. UploadManager:
   - 파일 선택 -> uploadFileApi 호출 -> (promise, cancel)반환
   - registerCancel(id, cancel)으로 취소 함수 저장
   - onAddMeta 로 Redux 스토더에 {id, name, size, progress} 메타데이터 전송

2. UploadModal:
   - Redux에서 uploadFiles 구독 -> 리스트 렌더링
   - cancelUpload(id) / cancelAllUploads() 호출 -> 실체 HTTP 요청 중단

## 결론

이렇게 Axios AbortController + 모듈 싱클톤 서비스의 조합으로, 대용량 파일 업로드 중 언제든 취소할 수 있습니다.


