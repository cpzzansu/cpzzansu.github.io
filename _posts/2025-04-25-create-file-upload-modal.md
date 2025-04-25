---
title: 끊김 없는 파일 업로드를 위한 React 최상단 Modal 배치하기
description: >-
  React + Redux 기반의 최상단 모달 구조를 통해, 화면 어디로 이동해도 끊김 없이 업로드 상황을 관리하는 방법을 소개합니다.
author: author_js
date: 2025-04-25 12:01:00 +0900
categories: [React, Cloud Storage]
tags: [html, react, javascript, css, file, upload, modal, redux]
pin: true
---

# 끊김 없는 파일 업로드를 위한 최상단 모달 컴포넌트 구현하기

## 1. 들어가며

최근 협업툴 개발 과정에서 Cloud Storage 기능을 추가했는데, 대용량 파일을 업로드할 때 업로드가 완료될 때까지 다른 작업을 진행할 수 없는 문제가 있었습니다. 이를 해결하기 위해, 다른 작업을 하면서도 파일 업로드가 끊기지 않도록 UI 구조를 변경하고 업로드 상태를 전역에서 관리하도록 개선했습니다.

이 글에서는 Redux를 활용한 상태 관리와 React를 통한 UI 구현을 중심으로 그 과정을 정리해 보겠습니다.

---

## 2. 전체 아키텍처 개요

이번 구현에서 핵심 아이디어는 다음과 같습니다.

- **Redux로 전역 상태 관리**
    - `fileUploadOpen`: 모달의 열림/닫힘 상태 관리
    - `uploadFiles`: 업로드 중인 파일의 리스트 및 진행률 관리

- **Layout 컴포넌트에 모달 배치**
    - 화면의 최상단에 `<FileUploadModal />` 컴포넌트를 배치하여 언마운트되지 않도록 유지

- **업로드 컴포넌트 구현**
    - 파일 선택 시 Redux에 파일 메타정보 저장
    - Axios를 통한 파일 업로드 및 진행률 콜백 관리

### 컴포넌트 구조
```
App ──▶ Provider(Redux)
└─ Layout ──▶ Navigation
   ├─ Outlet (현재 페이지)
   └─ FileUploadModal (fileUploadOpen = true일 때만 렌더)
```

---

## 3. Redux Slice 작성하기

파일 업로드 상태를 관리하는 Redux Slice를 작성합니다.

```javascript
// redux/slice/fileUploadSlice.js
import { createSlice } from '@reduxjs/toolkit';

const fileUploadSlice = createSlice({
  name: 'fileUpload',
  initialState: {
    fileUploadOpen: false,
    uploadFiles: [], // [{ id, fileName, size, type, progress }]
  },
  reducers: {
    setFileUploadOpen(state, { payload }) {
      state.fileUploadOpen = payload;
    },
    addUploadFiles(state, { payload }) {
      state.uploadFiles.push(...payload);
    },
    updateUploadProgress(state, { payload: { id, progress } }) {
      const file = state.uploadFiles.find(f => f.id === id);
      if (file) file.progress = progress;
    },
    clearUploadFiles(state) {
      state.uploadFiles = [];
    },
  },
});

export const {
  setFileUploadOpen,
  addUploadFiles,
  updateUploadProgress,
  clearUploadFiles,
} = fileUploadSlice.actions;

export default fileUploadSlice.reducer;
```

- `fileUploadOpen`: 모달을 열고 닫는 상태
- `uploadFiles`: 업로드 중인 파일들의 메타데이터와 진행 상태 저장

---

## 4. Layout 컴포넌트에 모달 연결하기

모달이 항상 최상단에 유지될 수 있도록 Layout 컴포넌트에 배치합니다.

```jsx
// components/common/Layout.jsx
import React from 'react';
import { Outlet } from 'react-router-dom';
import { useSelector } from 'react-redux';
import FileUploadModal from '../upload/FileUploadModal';
import Navigation from './Navigation';

export default function Layout() {
  const fileUploadOpen = useSelector(state => state.fileUpload.fileUploadOpen);

  return (
    <div className="app-layout">
      <Navigation />
      <main>
        <Outlet />
      </main>
      {fileUploadOpen && <FileUploadModal />}
    </div>
  );
}
```

- Navigation과 Outlet 사이가 메인 콘텐츠 영역
- `fileUploadOpen` 상태가 true일 때만 모달이 렌더링되어 항상 최상단에 유지됩니다.

---

## 5. FileUploadModal 구현

실제 파일을 업로드하고 진행률을 표시하는 모달 컴포넌트를 구현합니다.

```jsx
// components/upload/FileUploadModal.jsx
import React, { useRef } from 'react';
import { useDispatch, useSelector } from 'react-redux';
import {
  setFileUploadOpen,
  addUploadFiles,
  updateUploadProgress,
} from '../../redux/slice/fileUploadSlice';
import { uploadFileApi } from '../../apis/drive/driveApi';

export default function FileUploadModal() {
  const dispatch = useDispatch();
  const uploadFiles = useSelector(state => state.fileUpload.uploadFiles);
  const fileInputRef = useRef();

  const handleFileChange = (e) => {
    dispatch(setFileUploadOpen(true));

    const files = Array.from(e.target.files);
    const newUploads = files.map(file => ({
      id: `${file.name}-${file.size}-${Date.now()}`,
      fileName: file.name,
      size: file.size,
      type: file.type,
      progress: 0,
    }));

    dispatch(addUploadFiles(newUploads));

    newUploads.forEach(({ id, fileName }) => {
      const formData = new FormData();
      formData.append('file', files.find(f => f.name === fileName));
      uploadFileApi({
        formData,
        setProgress: (p) => dispatch(updateUploadProgress({ id, progress: p })),
      });
    });
  };

  return (
    <div className="modal-backdrop">
      <div className="modal-content">
        <h2>파일 업로드</h2>
        <input
          ref={fileInputRef}
          type="file"
          multiple
          onChange={handleFileChange}
          style={{ display: 'none' }}
        />
        <button onClick={() => fileInputRef.current.click()}>파일 선택</button>
        <button onClick={() => dispatch(setFileUploadOpen(false))}>닫기</button>

        <ul className="upload-list">
          {uploadFiles.map(f => (
            <li key={f.id}>
              <span>{f.fileName}</span>
              <progress value={f.progress} max="100" />
            </li>
          ))}
        </ul>
      </div>
    </div>
  );
}
```

- Axios를 사용하여 파일 업로드 및 진행률 업데이트

---

## 6. API 호출 유틸

Axios를 활용한 업로드 유틸을 정의하여 진행률을 표시합니다.

```javascript
import axios from 'axios';

export const uploadFileApi = async ({ formData, setProgress }) => {
  try {
    await axios.post('/api/drive/upload', formData, {
      headers: { 'Content-Type': 'multipart/form-data' },
      onUploadProgress: e => setProgress(Math.round((e.loaded * 100) / e.total)),
    });
  } catch (err) {
    console.error('업로드 실패', err);
  }
};
```

---

## 7. 결론

이번 구현을 통해 페이지 전환, 탭 이동 등 브라우저 환경이 변경되더라도 파일 업로드가 중단되지 않고 안정적으로 진행될 수 있도록 하였습니다.



