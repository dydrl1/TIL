[Git] IntelliJ .idea 파일이 커밋되는 문제 해결
1. 문제 상황

프로젝트 작업 후 git status를 확인하면 아래와 같은 파일들이 계속 변경된 것으로 표시됨.

.idea/workspace.xml
.idea/compiler.xml
.idea/misc.xml
.idea/modules.xml
.idea/vcs.xml


특히 workspace.xml은 코드를 수정하지 않아도
IDE를 열기만 해도 변경되면서 불필요한 커밋 충돌과 노이즈를 발생시킴.

2. 원인 분석

IntelliJ의 .idea 디렉토리는 개인 개발 환경 설정 파일이다.

창 배치

실행 설정

로컬 SDK 경로

플러그인 상태

등이 포함되어 있어 팀원마다 다를 수밖에 없는 정보이며,
Git으로 관리하면 오히려 협업에 방해가 된다.

3. 해결 전략
1) .gitignore에 IntelliJ 설정 파일 추가

.gitignore에 다음 항목을 추가한다.

# IntelliJ IDEA
.idea/
*.iml

2) 이미 Git이 추적 중인 경우 – 캐시에서 제거

.gitignore만 추가해도
이미 커밋된 파일은 계속 추적된다.
따라서 Git 캐시에서 제거가 필요하다.

git rm -r --cached .idea


실제 파일은 삭제되지 않고
Git의 추적 대상에서만 제거된다.

4. 정리된 표준 절차
# 1. .gitignore 수정
vi .gitignore

# 2. IntelliJ 설정 파일 제거
git rm -r --cached .idea

# 3. 상태 확인
git status

# 4. 커밋
git commit -m "chore: remove IntelliJ .idea files from git tracking"

# 5. 푸시
git push origin dev

5. 정리
항목	내용
IDE 설정 파일	개인 환경 전용
Git 관리 여부	❌ 관리 대상 아님
문제점	불필요한 변경, 충돌, PR 노이즈
해결 방법	.gitignore + git rm --cached
6. 배운 점

Git에 올려야 할 파일과 올리면 안 되는 파일을 구분하는 기준이 중요하다.

IDE 설정 파일은 코드가 아니라 환경 정보이므로 버전 관리 대상이 아니다.

협업 환경에서는
“내 로컬에서만 의미 있는 파일”을 반드시 배제해야 한다.