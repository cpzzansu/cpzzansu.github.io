---
title: React + Spring Boot 기반 클라우드 스토리지 “파일 다운로드” 기능 구현
description: >-
  선택한 파일을 ZIP으로 묶어 한번에 내려받는 기능을 React와 Spring Boot로 구현해봤습니다. Blob처리, ZipOutputStream 스트리밍, HTTP 요청, 응답을 활용했습니다.
author: author_js
date: 2025-04-29 12:01:00 +0900
categories: [React, Cloud Storage]
tags: [html, react, javascript, css, file, upload, modal, redux, JAVA, SpringBoot]
pin: true
---

클라우드 스토리지 서비스를 개발하고 있습니다. 클라우드 스토리지 서비스에서 큰 핵심이라고 하면 파일 업로드와 파일 다운로드가 있을 텐데요.
이번 글이서는 그중 "파일 다운로드" 기능을 중심으로, React와 SpringBoot를 활용해서 구현했습니다.

## 1. 로직의 흐름

1. 사용자는 파일 리스트에서 체크박스로 다운로드 할 파일을 선택
2. 프론트엔드가 선택된 파일 ID 목록을 백엔드로 전송
3. 백엔드는 ZipOutputStream 으로 서버 측에서 실시간 압축 스트림 생성
4. 브라우저는 ZIP 바이너리를 Blob으로 받아 <a download>로 다운로드 트리거

이 구조를 활용하면, 한 번의 HTTP 요청으로 다수의 파일을 메모리 과다 없이 묶어서 전송할 수 있습니다.

## 2. React 프론트엔드 구현

### 2.1 선택된 파일 ID 관리

```jsx
// 파일 리스트 컴포넌트 내부
const [selectedIds, setSelectedIds] = useState([]);

// 체크박스 클릭 시 토글
const toggleSelect = (id) => {
  setSelectedIds(prev => {
    const next = new Set(prev);
    next.has(id) ? next.delete(id) : next.add(id);
    return next;
  });
};
```

### 2.2 ZIP 다운로드 요청 & Blob 처리

```jsx
const downloadSelected = async () => {
  if (!selectedIds.size) {
    return alert('파일을 하나 이상 선택하세요.');
  }

  try {
    // ZIP 바이너리(blob) 응답 받기
    const res = await axios.post(
      '/api/files/download-multiple',
      { fileIds: Array.from(selectedIds) },
      { responseType: 'blob' }
    );

    // Blob → Object URL
    const blob = new Blob([res.data], { type: 'application/zip' });
    const url  = URL.createObjectURL(blob);

    // 다운로드 링크 생성
    const a = document.createElement('a');
    a.href = url;
    a.download = `download-${new Date().toISOString().slice(0,19).replace(/[-:T]/g,'')}.zip`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  } catch (err) {
    console.error(err);
    alert('다운로드에 실패했습니다.');
  }
};
```
```jsx
<button onClick={downloadSelected}>
  선택 파일 다운로드
</button>
```

## 3. Spring Boot: 백엔드 컨트롤러

```java
@RestController
@RequestMapping("/api/files")
@RequiredArgsConstructor
public class FileController {
  private final FileService fileService;

  @PostMapping("/download-multiple")
  public void downloadMultiple(
      @RequestBody FileIdsDto dto,
      HttpServletResponse response
  ) throws IOException {
    response.setContentType("application/zip");
    response.setHeader(
      "Content-Disposition",
      "attachment; filename=\"selected-files.zip\""
    );
    fileService.streamFilesAsZip(dto.getFileIds(), response.getOutputStream());
  }
}

// DTO
@Data
public class FileIdsDto {
  private List<Long> fileIds;
}
```

- Content-Disposition 헤더로 클라이언트 다운로드를 강제합니다.

## 4. ZipOutputStream 으로 묶기

```java
@Service
@RequiredArgsConstructor
public class FileServiceImpl implements FileService {
  private final FileRepository fileRepo;

  @Override
  public void streamFilesAsZip(List<Long> ids, OutputStream os) throws IOException {
    try (ZipOutputStream zos = new ZipOutputStream(os)) {
      byte[] buffer = new byte[8192];

      for (Long id : ids) {
        FileEntity file = fileRepo.findById(id)
          .orElseThrow(() -> new EntityNotFoundException(id + " not found"));

        zos.putNextEntry(new ZipEntry(file.getFileName()));
        try (InputStream in = Files.newInputStream(Paths.get(file.getFilePath()))) {
          int len;
          while ((len = in.read(buffer)) != -1) {
            zos.write(buffer, 0, len);
          }
        }
        zos.closeEntry();
      }
      zos.finish();
    }
  }
}
```

- 서버 메모리에 ZIP 전체를 올리지 않고 스트리밍하기 때문에 대용량 파일도 무리 없이 처리 가능합니다.

## 결론

업로드 기능만큼 다운로드 기능도 클라우드 스토리지에서 중요한 부분입니다.
여기 정리한 React에서 Spring Boot 기반 ZIP 다운로드 구조를 바탕으로, 
필요에 따라 진행률 표시나 권한 검증, 대용량 처리 등도 개발을 할지 생각해봐야겠습니다.
