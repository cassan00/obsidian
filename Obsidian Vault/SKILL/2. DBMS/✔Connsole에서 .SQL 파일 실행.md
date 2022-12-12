#DBMS


리눅스 환경에서 특정경로에 있는 sql 파일을 실행시켜야 할때 사용

1) vi 편집기로 SQL 작성 (실행할 sql 파일이 있다면 생략)
2) cd 명령어를 이용해 해당 위치로 이동
3) 해당위치에서 DBMS접속명령 실행(sqlplus 또는 tbsql)
4) 계정접속 
5) @파일명.sql 실행
6) 결과값 확인


## 사용 vi 명령어
```vim

vi (파일명)  - 해당파일의 vi 편집기 실행. 공란으로 실행시 new file
:w (파일명)  - 파일명 입력시 새파일로 저장. 공란으로 실행시 일반저장. :w! 덮어쓰기
:q           - 종료. :q! 강제종료 
```


# 참고
https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=sophie_yeom&logNo=220811401399
