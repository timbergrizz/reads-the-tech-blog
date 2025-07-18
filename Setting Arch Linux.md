- 설치 관련
	- 라이브 부트 USB 만들고, USB로 부트
		- 이때 바이오스에서 Secure boot 켜져있으면 비활성화
	- 파티션 설정 하고,  archinstall command 사용해서 설치
		- {{video https://youtu.be/WaWB3F-ffcI?feature=shared}}
  - JaKooLit dotfile 설치
    - Arch Install에서 별도의 프로파일 설치하지 않고 (cli 환경으로 외부에서 실행하기 위한 ssh정도만 설치하면 ok) 바로 [JaKooLit Hyprland dot](https://github.com/JaKooLit/Hyprland-Dots) 설치
    - 엔비디아 드라이브도 패키지에 포함되어 설치됨
      - `nvidia-smi`로 정상 설치 확인 가능
    - ROG 관련 패키지 제외 extra 패키지 모두 설치

- 한글 입력 관련
	- fcitx5 설치 및 설정
		- ``` bash
		  sudo pacman -S fcitx5 fcitx5-hangul fcitx5-gtk fcitx5-qt fcitx5-configtool	
		  ```
	- 관련 이슈
		- 한자창 관련
			- Configure - Addons - Hangul - Hanja mode 비활성화 및 키매핑 삭제
			- 한자라는 기능 자체를 써본 적이 없으니 그냥 비활성화. 필요시 모드 활성화.
		- Electron 어플리케이션 입력기 문제
			- /usr/share/application/ 에 설정되어 있는 Desktop 파일의 Execute 명령 다음과 같이 수정
				- ```
				  Exec = logseq %U --enable-wayland-ime --ozone-platform=wayland
				  ```
				- ime를 wayland ime (fcitx) 로 변경하는 커맨드

- 설치한 패키지들
	- 브라우저 : Zen browser
		- pacman 구조상 의존성인 파이어폭스부터 빌드하는거라 빌드 테스트까지 2시간 걸림. (Ultra7 155H)
	- IDE : Jetbrains Toolbox (IntelliJ, Datagrip), gemini cli, Zed
		- Jetbrains 플러그인으로 Windsurf 설치
		- Envfile 설정
      - Java에서 .env 사용하는데, 어플리케이션에서 파일 로드하는게 아닌, Run 옵션에서 실행하도록 하는 옵션 
	- 기타 : Logseq, Mattermost
	- CLI : fastfetch, bottom, neovim, docker, kubectl, telepresence
	- 터미널 : Kitty / agnoster (Multi-line 추가 설정)
    - 결국 [p10k](https://github.com/romkatv/powerlevel10k) 깜

- 스크롤 반대로 (맥 자연스러운 스크롤링?)
	- User Setting - Input.Native Scroll true로 설정

- 추가 세팅
	- WM 조작 키 관련 설정
		- hjkl로 방향키 대체. 이때 j가 아래다.
		- Swap이 윈도우끼리 이미 설정된 레이아웃 안에서 바꿔주는거라 Swap을 주로 쓰시면 되고, Change는 쓸 일 없을듯
	- Git 기반 Logseq Sync
		- [CharlesChiuGit/Logseq-Git-Sync-101](https://github.com/CharlesChiuGit/Logseq-Git-Sync-101)
		- Mac 부분 보고 따라했다.

- 테마 관련
	- IDE랑 에디터들은 Catputcchin Frappe로 맞춰뒀다.
  - 이후에 [Everforest](https://github.com/sainnhe/everforest) 로 변경. [nord](https://www.nordtheme.com/)랑 두개가 제일 마음에 듬.

- neovim 관련
  - 기존에 사용하던 dotfile을 못찾아 lazyvim 사용
