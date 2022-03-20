- 👋 Hi, I’m @a86860469
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...

<!---
a86860469/a86860469 is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
KB
 env :   #全局默認值
  PACKAGE_MANAGER_INSTALL : " apt-get update && apt-get install -y "
  MAKEJOBS：“ -j10 ”
  TEST_RUNNER_PORT_MIN : " 14000 "   #必須大於 12321，用於 http 緩存。見 https://cirrus-ci.org/guide/writing-tasks/#http-cache
  CCACHE_SIZE：“ 200M ”
  CCACHE_DIR：“ /tmp/ccache_dir ”
  CCACHE_NOHASHDIR : " 1 "   #如果構建目錄發生變化，調試信息可能包含陳舊的路徑，但這很好

cirrus_ephemeral_worker_template_env : &CIRRUS_EPHEMERAL_WORKER_TEMPLATE_ENV
  DANGER_RUN_CI_ON_HOST : " 1 "   #容器運行後會被丟棄，所以不存在ci腳本修改系統的風險

persistent_worker_template_env : &PERSISTENT_WORKER_TEMPLATE_ENV
  RESTART_CI_DOCKER_BEFORE_RUN：“ 1 ”

persistent_worker_template : &PERSISTENT_WORKER_TEMPLATE
  persistent_worker : {}   # https://cirrus-ci.org/guide/persistent-workers/

# https://cirrus-ci.org/guide/tips-and-tricks/#sharing-configuration-between-tasks
filter_template : &FILTER_TEMPLATE
  skip : $CIRRUS_REPO_FULL_NAME == "bitcoin-core/gui" && $CIRRUS_PR == ""   #不需要在只讀鏡像上運行，除非是 PR。https://cirrus-ci.org/guide/writing-tasks/#conditional-task-execution
  有狀態的：false   # https://cirrus-ci.org/guide/writing-tasks/#stateful-tasks

base_template : &BASE_TEMPLATE
  << : *FILTER_TEMPLATE
  合併基礎腳本：
    #無條件安裝 git（在指紋腳本中使用）並設置
    #默認 git 作者姓名（在 verify-commits.py 中使用）
    - bash -c "$PACKAGE_MANAGER_INSTALL git"
    - git config --global user.email "ci@ci.ci"
    - git config --global user.name "ci"
    -如果 [ "$CIRRUS_PR" = "" ]; 然後退出0；菲
    - git fetch $CIRRUS_REPO_CLONE_URL $CIRRUS_BASE_BRANCH
    - git merge FETCH_HEAD   #合併基礎以檢測靜默合併衝突

主模板：& MAIN_TEMPLATE
  timeout_in : 120m   # https://cirrus-ci.org/faq/#instance-timed-out
  容器：
    # https://cirrus-ci.org/faq/#are-there-any-limits
    #每個項目一共16個CPU，每個容器分配2個，這樣8個任務並行運行
    中央處理器：2
    貪心：真的
    memory : 8G   #設置為 8GB 以避免 OOM。https://cirrus-ci.org/guide/linux/#linux-containers
  ccache_cache：
    文件夾：“ /tmp/ccache_dir ”
  依賴內置緩存：
    文件夾：“依賴/構建”
    指紋腳本：回顯 $CIRRUS_TASK_NAME $(git rev-list -1 HEAD ./depends)
  ci_script：
    - ./ci/test_run_all.sh

global_task_template : &GLOBAL_TASK_TEMPLATE
  << : *BASE_TEMPLATE
  << : *MAIN_TEMPLATE

計算信用模板：&信用模板
  # https://cirrus-ci.org/pricing/#compute-credits
  #只對主倉庫的拉取請求使用積分
  use_compute_credits : $CIRRUS_REPO_FULL_NAME == '比特幣/比特幣' && $CIRRUS_PR != ""

任務：
  名稱：' lint [仿生] '
  << : *BASE_TEMPLATE
  容器：
    image : ubuntu:bionic   #對於 python 3.6，根據 doc/dependencies.md 支持的最舊版本
    中央處理器：1
    內存：1G
  #為了更快的 CI 反饋，立即安排 linter
  << : *CREDITS_TEMPLATE
  lint_script：
    - ./ci/lint_run_all.sh
  環境：
    << : *CIRRUS_EPHEMERAL_WORKER_TEMPLATE_ENV

任務：
  名稱：“ Win64 原生 [msvc] ”
  << : *FILTER_TEMPLATE
  windows_container：
    中央處理器：4
    內存：8G
    圖片：cirrusci/windowsservercore:visualstudio2019
  timeout_in : 120m
  環境：
    路徑：' C:\jom;C:\Python39;C:\Python39\Scripts;C:\Program Files (x86)\Microsoft Visual Studio\2019\BuildTools\MSBuild\Current\Bin;%PATH% '
    PYTHONUTF8：1
    CI_VCPKG_TAG：' 2022.02.23 '
    VCPKG_DOWNLOADS：' C:\Users\ContainerAdministrator\AppData\Local\vcpkg\downloads '
    VCPKG_DEFAULT_BINARY_CACHE：' C:\Users\ContainerAdministrator\AppData\Local\vcpkg\archives '
    QT_DOWNLOAD_URL：' https ://download.qt.io/official_releases/qt/5.15/5.15.2/single/qt-everywhere-src-5.15.2.zip '
    QT_LOCAL_PATH : ' C:\qt-everywhere-src-5.15.2.zip '
    QT_SOURCE_DIR：' C:\qt-everywhere-src-5.15.2 '
    QTBASEDIR：' C:\Qt_static '
    x64_NATIVE_TOOLS : ' "C:\Program Files (x86)\Microsoft Visual Studio\2019\BuildTools\VC\Auxiliary\Build\vcvars64.bat" '
    IgnoreWarnIntDirInTempDetected：“真”
  合併腳本：
    - git config --global user.email "ci@ci.ci"
    - git config --global user.name "ci"
    # Windows 文件系統丟失可執行位，以及所有可執行文件
    #文件現在被視為“修改”。它將破壞以下 `git merge`
    #命令。接下來的兩個命令讓 git 忽略這個問題。
    - git config core.filemode 假
    -混帳重置--硬
    - PowerShell -NoLogo -Command if ($env:CIRRUS_PR -ne $null) { git fetch $env:CIRRUS_REPO_CLONE_URL $env:CIRRUS_BASE_BRANCH; git 合併 FETCH_HEAD；}
  msvc_qt_built_cache：
    文件夾：“ %QTBASEDIR% ”
    重新上傳_on_changes：假
    指紋腳本：
      -回顯 %QT_DOWNLOAD_URL%
-msbuild       -版本
    填充腳本：
      - curl -L -o C:\jom.zip http://download.qt.io/official_releases/jom/jom.zip
      - mkdir C:\jom
      -tar -xf C:\jom.zip -CC:\jom
      - curl -L -o %QT_LOCAL_PATH% %QT_DOWNLOAD_URL%
      -tar -xf %QT_LOCAL_PATH% -CC:\
      - ' %x64_NATIVE_TOOLS% '
      - cd %QT_SOURCE_DIR%
      - mkdir 構建
      -光盤構建
      - ..\configure -release -silent -opensource -confirm-license -opengl desktop -static -static-runtime -mp -qt-zlib -qt-pcre -qt-libpng -nomake 示例 -nomake 測試 -nomake 工具 -no-angle -no -dbus -no-gif -no-gtk -no-ico -no-icu -no-libjpeg -no-libudev -no-sql-sqlite -no-sql-odbc -no-sqlite -no-vulkan -skip qt3d -跳過 qtactiveqt -skip qtandroidextras -skip qtcharts -skip qtconnectivity -skip qtdatavis3d -skip qtdeclarative -skip doc -skip qtdoc -skip qtgamepad -skip qtgraphicaleffects -skip qtimageformats -skip qtlocation -skip qtlottie -skip qtmacextras -skip qtmultimedia -skip q -skip qtquick3d -skip qtquickcontrols -skip qtquickcontrols2 -skip qtquicktimeline -skip qtremoteobjects -skip qtscript -skip qtscxml -skip qtsensors -skip qtserialbus -skip qtserialport -skip qtspeech -skip qtsvg -skip qtvirtualkeyboard -skipqtwayland -skip qtwebchannel -skip qtwebengine -skip qtwebglplugin -skip qtwebsockets -skip qtwebview -skip qtx11extras -skip qtxmlpatterns -no-openssl -no-feature-bearermanagement -no-feature-printdialog -no-feature-printer -no-feature-printpreviewdialog -no-feature-printpreviewwidget -no-feature-sql -no-feature-sqlmodel -no-feature-textbrowser -no-feature-textmarkdownwriter -no-feature-textodfwriter -no-feature-xml -prefix %QTBASEDIR%-no-feature-xml -prefix %QTBASEDIR%-no-feature-xml -prefix %QTBASEDIR%
      -喬姆
      -喬姆安裝
  vcpkg_tools_cache：
    文件夾：“ %VCPKG_DOWNLOADS%\tools ”
    重新上傳_on_changes：假
    指紋腳本：
      -回顯 %CI_VCPKG_TAG%
-msbuild       -版本
  vcpkg_binary_cache：
    文件夾：“ %VCPKG_DEFAULT_BINARY_CACHE% ”
    重新上傳_on_changes：真
    指紋腳本：
      -回顯 %CI_VCPKG_TAG%
      -類型 build_msvc\vcpkg.json
-msbuild       -版本
    填充腳本：
      - mkdir %VCPKG_DEFAULT_BINARY_CACHE%
  install_python_script：
    - choco install --yes --no-progress python3 --version=3.9.6
    -點安裝 zmq
    -蟒蛇-VV
  install_vcpkg_script：
    -光盤..
    - git clone --quiet https://github.com/microsoft/vcpkg.git
    - cd vcpkg
    - git -c advice.detachedHead=false checkout %CI_VCPKG_TAG%
    - .\bootstrap-vcpkg -disableMetrics
    -迴聲集（VCPKG_BUILD_TYPE 版本）>> 三元組\x64-windows-static.cmake
    - .\vcpkg 集成安裝
    - .\vcpkg 版本
  構建腳本：
    - cd %CIRRUS_WORKING_DIR%
    - python build_msvc\msvc-autogen.py
    - msbuild build_msvc\bitcoin.sln -property:Configuration=Release -maxCpuCount -verbosity:minimal -noLogo
  單元測試腳本：
    - src\test_bitcoin.exe -l test_suite
    - src\bench_bitcoin.exe > NUL
    - python 測試\util\test_runner.py
    -python測試\util\rpcauth-test.py
  功能測試腳本：
    #將動態端口範圍增加到最大允許值以緩解“OSError: [WinError 10048] 每個套接字地址（協議/網絡地址/端口）通常只允許使用一次”。
    #請參閱：https://docs.microsoft.com/en-us/biztalk/technical-guides/settings-that-c​​an-be-modified-to-improve-network-performance
    - netsh int ipv4 設置動態端口 tcp start=1025 num=64511
    - netsh int ipv6 set dynamicport tcp start=1025 num=64511
    #由於超時，暫時排除 feature_dbcrash
    - python test\functional\test_runner.py --nocleanup --ci --quiet --combinedlogslen=4000 --jobs=4 --timeout-factor=8 --extended --exclude feature_dbcrash

任務：
  name : ' ARM [單元測試，無功能測試] [bullseye] '
  << : *GLOBAL_TASK_TEMPLATE
  arm_container：
    圖片：debian：bullseye
    中央處理器：2
    內存：8G
  環境：
    << : *CIRRUS_EPHEMERAL_WORKER_TEMPLATE_ENV
    FILE_ENV：“ ./ci/test/ 00_setup_env_arm.sh ”
    QEMU_USER_CMD : " "   #禁用 qemu 並原生運行測試

任務：
  name : ' Win64 [單元測試，無 gui 測試，無 boost::process，無功能測試] [jammy] '
  << : *GLOBAL_TASK_TEMPLATE
  容器：
    圖片：ubuntu：jammy
  環境：
    << : *CIRRUS_EPHEMERAL_WORKER_TEMPLATE_ENV
    FILE_ENV：“ ./ci/test/ 00_setup_env_win64.sh ”

任務：
  名稱：'32 位 + 破折號 [gui] [CentOS 8 ] '
  << : *GLOBAL_TASK_TEMPLATE
  容器：
    圖片： quay.io/centos/centos: stream8
  環境：
    << : *CIRRUS_EPHEMERAL_WORKER_TEMPLATE_ENV
    PACKAGE_MANAGER_INSTALL：“百勝安裝-y ”
    FILE_ENV：“ ./ci/test/ 00_setup_env_i686_centos.sh ”

任務：
  name : ' [以前的版本，使用 qt5 開發包和一些依賴包，DEBUG] [unsigned char] [buster] '
  previous_releases_cache：
    文件夾：“發布”
  << : *GLOBAL_TASK_TEMPLATE
  << : *PERSISTENT_WORKER_TEMPLATE
  環境：
    << : *PERSISTENT_WORKER_TEMPLATE_ENV
    FILE_ENV：“ ./ci/test/ 00_setup_env_native_qt5.sh ”

任務：
  名稱：' [TSan，取決於，gui] [jammy] '
  << : *GLOBAL_TASK_TEMPLATE
  容器：
    圖片：ubuntu：jammy
    cpu : 6   #增加 CPU 和內存以避免超時
    內存：24G
  環境：
    << : *CIRRUS_EPHEMERAL_WORKER_TEMPLATE_ENV
    FILE_ENV：“ ./ci/test/ 00_setup_env_native_tsan.sh ”

任務：
  名稱：' [MSan，取決於] [焦點] '
  << : *GLOBAL_TASK_TEMPLATE
  容器：
    圖片：ubuntu：焦點
  環境：
    << : *CIRRUS_EPHEMERAL_WORKER_TEMPLATE_ENV
    FILE_ENV：“ ./ci/test/ 00_setup_env_native_msan.sh ”
    MAKEJOBS : " -j4 "   #避免由於 MSan 導致的過多內存使用

任務：
  name : ' [ASan + LSan + UBSan + 整數，不依賴於] [jammy] '
  << : *GLOBAL_TASK_TEMPLATE
  容器：
    圖片：ubuntu：jammy
  環境：
    << : *CIRRUS_EPHEMERAL_WORKER_TEMPLATE_ENV
    FILE_ENV：“ ./ci/test/ 00_setup_env_native_asan.sh ”
    MAKEJOBS : " -j4 "   #避免過度使用內存

任務：
  name : ' [fuzzer,address,undefined,integer, no depends] [jammy] '
  << : *GLOBAL_TASK_TEMPLATE
  容器：
    圖片：ubuntu：jammy
    cpu : 4   #增加 CPU 和內存以避免超時
    內存：16G
  環境：
    << : *CIRRUS_EPHEMERAL_WORKER_TEMPLATE_ENV
    FILE_ENV：“ ./ci/test/ 00_setup_env_native_fuzz.sh ”

任務：
  名稱：' [多進程，i686，調試] [焦點] '
  << : *GLOBAL_TASK_TEMPLATE
  容器：
    圖片：ubuntu：焦點
    中央處理器：4
    memory : 16G   #默認內存有時有點小，所以加倍
  環境：
    << : *CIRRUS_EPHEMERAL_WORKER_TEMPLATE_ENV
    FILE_ENV：“ ./ci/test/ 00_setup_env_i686_multiprocess.sh ”

任務：
  name : ' [沒有錢包，libbitcoinkernel] [bionic] '
  << : *GLOBAL_TASK_TEMPLATE
  容器：
    圖片：ubuntu：仿生
  環境：
    << : *CIRRUS_EPHEMERAL_WORKER_TEMPLATE_ENV
    FILE_ENV：“ ./ci/test/ 00_setup_env_native_nowallet_libbitcoinkernel.sh ”

任務：
  名稱：' macOS 10.15 [gui，無測試] [焦點] '
  << : *BASE_TEMPLATE
  macos_sdk_cache：
    文件夾：“依賴/SDKs/$MACOS_SDK ”
    指紋密鑰：“ $MACOS_SDK ”
  << : *MAIN_TEMPLATE
  容器：
    圖片：ubuntu：焦點
  環境：
    MACOS_SDK：“ Xcode-12.2-12B45b-extracted-SDK-with-libcxx-headers ”
    << : *CIRRUS_EPHEMERAL_WORKER_TEMPLATE_ENV
    FILE_ENV：“ ./ci/test/ 00_setup_env_mac.sh ”

任務：
  name : ' macOS 12 本機 [gui，僅限系統 sqlite] [不依賴於] '
  brew_install_script：
    - brew install boost libevent qt@5 miniupnpc libnatpmp ccache zeromq qrencode libtool automake gnu-getopt
  << : *GLOBAL_TASK_TEMPLATE
  macos_instance：
    #使用最新的圖像，但硬編碼版本以避免靜默升級（和中斷）
    圖片：monterey-xcode-13.2   # https://cirrus-ci.org/guide/macOS
  環境：
    << : *CIRRUS_EPHEMERAL_WORKER_TEMPLATE_ENV
    CI_USE_APT_INSTALL：“沒有”
    PACKAGE_MANAGER_INSTALL : " echo "   #無事可做
    FILE_ENV：“ ./ci/test/ 00_setup_env_mac_host.sh ”

任務：
  名稱：' ARM64 Android APK [焦點] '
  << : *BASE_TEMPLATE
  android_sdk_cache：
    文件夾：“依賴/SDKs/android ”
    指紋密鑰：“ ANDROID_API_LEVEL=28 ANDROID_BUILD_TOOLS_VERSION=28.0.3 ANDROID_NDK_VERSION=23.1.7779620 ”
  依賴源緩存：
    文件夾：“依賴/來源”
    指紋腳本：git rev-list -1 HEAD ./depends
  << : *MAIN_TEMPLATE
  容器：
    圖片：ubuntu：焦點
  環境：
    << : *CIRRUS_EPHEMERAL_WORKER_TEMPLATE_ENV
    FILE_ENV：“ ./ci/test/ 00_setup_env_android.sh ”
