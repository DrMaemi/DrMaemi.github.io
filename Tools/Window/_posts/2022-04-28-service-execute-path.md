---
title: '[Window] 서비스(Services) 실행 경로'
author_profile: true
toc_label: '[Window] 서비스(Services) 실행 경로'
---

윈도우는 서비스(프로그램)들을 관리하는 시스템인 Services가 있다.

## 서비스 실행 경로 수정
로컬에 MySQL을 설치한 후 설치된 경로를 변경하고 환경 변수 편집을 완료해서 잘 개발하다가, PC를 재부팅하고 로컬에 MySQL 서버가 구동되지 않아 서비스 실행을 시도했는데 실행 경로가 예전 설치 경로로 지정되어 있어 오류가 발생했다.

서비스 실행 경로를 수정하는 방법은 레지스트리 편집기를 이용해서 할 수 있다.

1. <svg width="4%" height="auto" role="img" viewBox="0 0 24 24" style="float:left;" xmlns="http://www.w3.org/2000/svg"><path d="M0 3.449L9.75 2.1v9.451H0m10.949-9.602L24 0v11.4H10.949M0 12.6h9.75v9.451L0 20.699M10.949 12.6H24V24l-12.9-1.801"/></svg>+`r`로 실행 명령창을 열어 `regedit` 입력 후 실행
2. HKEY_LOCAL_MACHINE > SYSTEM > CurrentControlSet > Services 에서 원하는 서비스 명 선택 후 `ImagePath` 변수 수정

## A. 참조
잡동사니 IT 저장소, "윈도우 서비스(Services) 실행 파일 경로 수정", *Tistory*, Aug. 24, 2020. [Online]. Available: [https://itwarehouses.tistory.com/14](https://itwarehouses.tistory.com/14) [Accessed Apr. 28, 2022].
