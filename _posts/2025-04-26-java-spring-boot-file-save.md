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
- 로그로 파일명과 폴더 ID를 기록하여 업로드 확인을 쉽게 합니다.

## 2. FileService 인터페이스 구성

서비스의 추상화를 위해 FileService 인터페이스를 만들어 두면 유영한 구현체 교체가 가능합니다.

FileService.java
```java
public interface FileService {  
    void uploadFile(MultipartFile file, Long folderId) throws IOException;  
}
```

## 3. FileService 구현체 작성하기

구체적인 파일 저장 로직을 담은 구현체입니다. 파일을 서버에 저장하고 메타데이터를 데이터베이스에 저장합니다.

FileServiceImpl.java
```java
@Service
@RequiredArgsConstructor
public class FileServiceImpl implements FileService {

    @Value("${app.upload.dir}")  
    private String uploadDir;  
  
    private final FolderRepository folderRepository;  
    private final FileRepository fileRepository;

    @Override  
    public void uploadFile(MultipartFile file, Long folderId) throws IOException {  

        // 1) 폴더 존재 여부 확인
        Folder folder = folderRepository.findById(folderId)  
            .orElseThrow(() -> new EntityNotFoundException("폴더가 존재하지 않습니다: " + folderId));  

        // 2) 날짜 기반 디렉토리 구성
        String yearMonth = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy/MM"));  
        Path targetLocation = Paths.get(uploadDir).resolve(yearMonth);
        Files.createDirectories(targetLocation);  // 디렉토리가 없으면 자동 생성합니다.

        // 3) 고유 파일명 생성 (중복 방지)
        String original = StringUtils.cleanPath(file.getOriginalFilename());  
        String fileName = UUID.randomUUID() + "_" + original;

        // 4) 서버에 파일 복사 (저장)
        Path destination = targetLocation.resolve(fileName);  
        try (InputStream in = file.getInputStream()) {  
            Files.copy(in, destination, StandardCopyOption.REPLACE_EXISTING);  
        }

        // 5) DB에 파일 메타데이터 저장
        File entity = File.builder()
                .fileName(original)
                .contentType(file.getContentType())
                .size(file.getSize())
                .filePath(destination.toString())
                .uploadedAt(LocalDateTime.now())
                .folder(folder)
                .build();

        fileRepository.save(entity);  
    }
}
```

💡코드 분석
- 폴더 존재 확인: 업로드할 대상 폴더가 DB에 없으면 예외 발생
- 날짜 기반 디렉토리: 업로드된 날짜 기준으로 연도/월(yyyy/MM) 형태로 폴더를 자동 구성합니다.
- UUID를 활용한 고유 파일명: 파일 이름 충돌을 방지하기 위해 UUID를 앞에 추가하여 유니크한 파일명을 생성합니다.
- 파일저장: Java NIO의 Files.copy()를 사용하여 실제 서버의 디렉토리에 파일을 저장합니다.
- 메타데이터 DB 저장: 저장된 파일의 경로, 크기, 이름 등의 메타 정보를 DB에 저장해 관리합니다.

## 4. 관련 설정(application.yml)

파일이 저장될 기본 경로를 설정합니다.

```yaml
app:
  upload:
    dir: "/var/www/myapp/uploads"
```

## 5. 데이터베이스 구조

File 테이블의 JPA 엔티티 예시입니다.
```java
@Entity
@Getter @Builder
@AllArgsConstructor @NoArgsConstructor(access = AccessLevel.PROTECTED)
public class File {

    @Id @GeneratedValue
    private Long id;

    private String fileName;
    private String contentType;
    private Long size;
    private String filePath;
    private LocalDateTime uploadedAt;

    @ManyToOne(fetch = FetchType.LAZY)
    private Folder folder;
}
```
- filePath에 실제 파일의 위치를 기록합니다.
- Folder 엔티티와의 관계로 특정 폴더 아래 파일을 관리할 수 있습니다.

## 6. 결론

이번 포스팅에서는 Cloud Storage 서비스를 구현 중 클라이언트에서 업로드 파일을 전달받아 서버에 저장하고, database에 기록하는 API Method 구조에
대해서 작성해봤습니다. 아직까지는 어려운 부분은 없었습니다. 물론 더 세세한 부분들을 추가하면서 생각치 못한 오류들을 만날 수 있겠지만, 아직은 순항중!~

