로컬 환경에서 프로젝트를 컴파일하고 참고 문서를 보려면, 아래의 명령들을 차례로 실행하세요:

$ npm install gitbook-cli

(package.json 파일이 없다는 경고가 표시 될 수 있지만, 무시해도 됩니다.)

$ ./node_modules/gitbook-cli/bin/gitbook.js install

위의 명령어를 통해 필요한 플러그인들과 최신 버전의 gitbook을 설치할 수 있습니다.

$ ./node_modules/gitbook-cli/bin/gitbook.js build

위의 명령어를 실행하면, _book 폴더로 문서를 빌드하게 됩니다.

위 명령어 대신 아래의 명령어를 실행 할 수 있습니다.

$ ./node_modules/gitbook-cli/bin/gitbook.js serve

4000번 포트로 웹 서버를 실행하며, md 확장자를 가지는 파일들을 모니터링하여(새로운 파일이 추가되거나 기존 파일을 변경을 감시),
변경이 있을 때 마다 다시 빌드를 합니다. 그리고 브라우져를 새로고침하여 변경 내용을 브라우져에서 확인 할 수 있습니다.
