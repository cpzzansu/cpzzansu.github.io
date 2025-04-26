---
title: 자바(Spring Boot)를 활용한 클라우드 스토리지 파일 저장 구현하기
description: >-
  클라우드 스토리지 서비스를 개발하면서 핵심적인 기능 중 하나는 바로 파일 업로드입니다. 이번 글에서는 자바(Spring Boot) 환경에서 파일 업로드 API와 이를 서버에 저장하는 방식을 실무 예제를 통해 자세히 설명하겠습니다.
author: author_js
date: 2025-04-25 12:01:00 +0900
categories: [React, Cloud Storage]
tags: [html, react, javascript, css, file, upload, modal, redux]
pin: true
---

## 1. API Controller 만들기

서버에서 파일 업로드를 처리하는 첫번째 관문은 Controller Method 입니다. 클라이언트에서 파일과 폴더 ID 값을 전달하면 이를 받아서 서비스로 전달합니다.

FileController.java
```java
@RestController
@RequestMapping("/api/files")
@RequiredArgsConstructor
public class FileController {

    private final FileService fileService;

    @PostMapping("/uploadFile")  
    public void uploadFile(
            @RequestParam(name = "file") MultipartFile file,
            @RequestParam(name = "folderId") Long folderId) throws IOException {

        log.debug("업로드된 파일: {}, 폴더 ID: {}", file.getOriginalFilename(), folderId);
        fileService.uploadFile(file, folderId);
    }
}
```
- @RequestParam으로 MultipartFile 객체와 폴더 ID를 전달받습니다.