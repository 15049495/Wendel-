
executor:
  name: linux-docker
  size: medium
steps:
  - checkout:
      path: src/electron
  - run:
      name: Setup third_party Depot Tools
      command: |
        # "depot_tools" has to be checkout into "//third_party/depot_tools" so pylint.py can a "pylintrc" file.
        git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git src/third_party/depot_tools
        echo 'export PATH="$PATH:'"$PWD"'/src/third_party/depot_tools"' >> $BASH_ENV
  - run:
      name: Download GN Binary
      command: |
        chromium_revision="$(grep -A1 chromium_version src/electron/DEPS | tr -d '\n' | cut -d\' -f4)"
        gn_version="$(curl -sL "https://chromium.googlesource.com/chromium/src/+/${chromium_revision}/DEPS?format=TEXT" | base64 -d | grep gn_version | head -n1 | cut -d\' -f4)"
        cipd ensure -ensure-file - -root . \<<-CIPD
        \$ServiceURL https://chrome-infra-packages.appspot.com/
        @Subdir src/buildtools/linux64
        gn/gn/linux-amd64 $gn_version
        CIPD
        echo 'export CHROMIUM_BUILDTOOLS_PATH="'"$PWD"'/src/buildtools"' >> $BASH_ENV
  - run:
      name: Download clang-format Binary
      command: |
        chromium_revision="$(grep -A1 chromium_version src/electron/DEPS | tr -d '\n' | cut -d\' -f4)"
        sha1_path='buildtools/linux64/clang-format.sha1'
        curl -sL "https://chromium.googlesource.com/chromium/src/+/${chromium_revision}/${sha1_path}?format=TEXT" | base64 -d > "src/${sha1_path}"
        download_from_google_storage.py --no_resume --no_auth --bucket chromium-clang-format -s "src/${sha1_path}"
  - run:
      name: Run Lint
      command: |
        # gn.py tries to find a gclient root folder starting from the current dir.
        # When it fails and returns "None" path, the whole script fails. Let's "fix" it.
        touch .gclient
        # Another option would be to checkout "buildtools" inside the Electron checkout,
        # but then we would lint its contents (at least gn format), and it doesn't pass it.
        cd src/electron
        node script/yarn install --frozen-lockfile
        node script/yarn lint
  - run:
      name: Run Script Typechecker
      command: |
        cd src/electron
        node script/yarn tsc -p tsconfig.script.json
