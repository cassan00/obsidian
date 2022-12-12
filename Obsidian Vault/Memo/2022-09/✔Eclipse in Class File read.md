


# 0. index 
이클립스에서 Class 파일을 열어보는 법은 크게 3가지가 있다.
 - JadClipse 
 - Enhanced Class Decompiler
 - JD-Eclipse

원래는 JadClipse 나 JadClipse+Class Decompiler 로 사용하는데, 새 플러그인을 찾아서 방법을 기록해놓는다. (JDK1.8에 Eclipse-Photon 환경에서 JadClipse가 동작을 안해서 새로 찾았다.) 세 가지 방법 중 한 가지만 선택해서 하면 된다. 



# 1. JadClipse
## 1.1. jad.exe 설치
[https://varaneckas.com/jad/](https://varaneckas.com/jad/)
-   파일명 : jad158g.win.zip > jad.exe
-   파일이동: `D:\jad` (경로상관없음)

## 1.2. jadClipse 설치
[jadclipse/](https://sourceforge.net/projects/jadclipse/)
-   파일명 : net.sf.jadclipse\_3.3.0.jar
-   파일이동 : `D:\eclipse\plugins`

## 1.3. Eclipse Setting
1) JadClipse Setting 
-   TopMenu > Window > Preferences > Java > JadClipse 메뉴 추가 확인
-   JadClipse - Path to decompiler : 1. 에서 다운받은 exe 파일의 전체경로 입력
2) Editors Setting
-   TopMenu > Window > Preferences > General > Editors > File Associations 
-   File types : class, class without source 선택 
-   Asscodiated editors : Add > JadClipse Class File Viewer > Default 


# 2. Enhanced Class Decompiler
https://marketplace.eclipse.org/content/enhanced-class-decompiler
-   TopMenu > Help > Eclipse Marketplace : class decompiler 검색 설치

# 3. JD-Eclipse
http://java-decompiler.github.io/
1) JD-Eclipse 설치

2) Editors Setting
-   TopMenu > Window > Preferences > General > Editors > File Associations 
-   File types : class, class without source 선택 
-   Asscodiated editors : Add > JadClipse Class File Viewer > Default 
-   Asscodiated editors : Add > JD Class File Viewer > Default 선택 


JD-Eclipse




https://marketplace.eclipse.org/content/enhanced-class-decompiler

http://java-decompiler.github.io/
