# NAS Player (tvOS) — LGPL 재링크 자료

App Store 의 Apple TV 앱 **NAS Player**(`com.nasplayer.tv`)는 LGPL 라이브러리를 씁니다.
이 저장소는 LGPL-2.1 §6 · LGPL-3.0 §4 에 따라, 사용자가 **수정한 라이브러리로 앱을 다시 링크**할 수 있게 하는 자료를 공개합니다.

- 앱 자체의 소스 코드는 공개하지 않습니다. 대신 앱의 **오브젝트 파일(.o)** 과 빌드 때의 **최종 링크 명령**을 공개합니다.
- App Store 에 올린 빌드마다 [릴리스](https://github.com/yjinchoi/nasplayer-relink/releases) `build-<빌드 번호>` 에 묶음(zip)을 올립니다.
  설치한 앱의 빌드 번호(「설정」의 버전 괄호 안)와 같은 릴리스를 받으세요. 「설정 → 오픈소스 라이선스」에 그 빌드의 릴리스 주소가 나옵니다.

## 묶음 구성

| 경로 | 내용 |
|---|---|
| `objects/app/` | 앱 자체 오브젝트(.o) |
| `objects/packages/` | 앱에 정적으로 들어간 비 LGPL Swift 패키지(SwiftNIO · Citadel · swift-crypto 등)의 오브젝트, MPVKit 의 Swift 껍데기 |
| `libs/` | 정적 라이브러리 **원본**(arm64만 남김) — MPVKit 구성 요소 28개 · MoltenVK · CocoaPods 묶음. 목록 `libs/LIST.txt` |
| `app/NASPlayer.app` | 실행 파일을 뺀 앱 묶음(리소스, TVVLCKit · AMSMB2 동적 프레임워크, 서명 · 프로파일 없음) |
| `NASPlayer.LinkFileList` · `link.sh` | 빌드 때 Xcode 가 쓴 최종 링크 명령(경로만 상대 경로로 바꿈) |
| `sign.sh` | 자기 인증서 · 프로파일로 서명 |
| `VERSIONS.txt` | Xcode · SDK 판, 라이브러리 판, git 커밋, 재링크 검증 결과 |

각 묶음은 올리기 전에 묶음만으로 다시 링크해, App Store 에 올린 실행 파일과 크기 · 심볼 · 링크한 dylib 이 같은지 자동으로 확인합니다(결과는 `VERSIONS.txt` 의 `relink-check`).

## 다시 링크하는 법

준비: macOS, `VERSIONS.txt` 의 `xcode:` 줄과 같은 판의 Xcode(예: 빌드 8 은 Xcode 27.0 (27A266a) · appletvos 27.0 SDK), Apple 개발자 계정.

1. **Xcode** — 같은 판을 설치하고 `sudo xcode-select -s /Applications/Xcode.app` 으로 고릅니다. 판이 다르면 `link.sh` 가 경고하며, 링크가 실패하거나 결과가 달라질 수 있습니다.
2. **라이브러리 바꾸기**
   - 정적 라이브러리(mpv · FFmpeg · libplacebo · gnutls 등): 아래 표의 소스와 MPVKit 빌드 스크립트로 수정본을 빌드해, `libs/` 의 같은 이름 파일 하나만 바꿉니다.
     예: `libs/Libmpv.framework/Libmpv` ← 수정본 `Libmpv.xcframework/tvos-arm64_arm64e/Libmpv.framework/Libmpv`(유니버설이어도 됩니다).
     공개 C API(심볼)가 호환돼야 링크됩니다.
   - 동적 프레임워크(TVVLCKit · AMSMB2): `app/NASPlayer.app/Frameworks/<이름>.framework` 를 수정본으로 바꿉니다.
3. **링크** — `./link.sh` → `app/NASPlayer.app/NASPlayer` 가 만들어집니다(arm64 tvOS 실행 파일).
4. **서명** — `./sign.sh "Apple Development: 이름 (팀ID)" 내프로파일.mobileprovision`
   - 프로파일은 자기 팀의 tvOS 앱 ID(와일드카드 가능)와 자기 Apple TV 를 넣어 developer.apple.com 에서 만듭니다. 번들 ID 는 프로파일의 것으로 바뀝니다.
   - 프로파일에 없는 권한(iCloud 키-값 저장소 · 키체인 공유)은 빠지므로 「설정 동기화」는 동작하지 않을 수 있습니다. 재생 기능에는 영향이 없습니다.
   - 무료 개인 팀(Personal Team)으로 Apple TV 에 설치할 수 있는지는 확인하지 못했습니다. 안 되면 유료 Apple Developer Program 이 필요합니다.
5. **설치** — Apple TV 를 Xcode 와 페어링한 뒤 `xcrun devicectl device install app --device <기기> app/NASPlayer.app`(또는 Xcode 의 Devices and Simulators 창에 끌어 놓기).

## 라이브러리 · 대응 소스

### MPVKit — 정적 링크

MPVKit [`1.1.0-n9.0.2`](https://github.com/mpvkit/MPVKit/tree/1.1.0-n9.0.2)(커밋 `98c3453e252922e3136422da3fbb2e341ed9edc8`, LGPL-3.0, 제품 `MPVKit` — GPL 제품은 쓰지 않음).
빌드 스크립트 `Sources/BuildScripts/XCFrameworkBuild/main.swift`, FFmpeg configure 인자(`ffmpegConfiguers` · `--enable-version3` · `--disable-shared --enable-static`),
mpv 패치 `Sources/BuildScripts/patch/libmpv/` 3개(`0001-player-add-moltenvk-context` · `0002-revert-build-static` · `0003-enable-avfoundation-ao-tvos`)가 이 태그에 있습니다.
의존 라이브러리는 아래 `mpvkit/*-build` 저장소의 태그로 빌드되며, 각 저장소의 `main.swift` 에 upstream 판이 적혀 있습니다.

| 구성 요소 (앱에 링크된 라이브러리) | upstream 판 | 라이선스 | upstream 소스 | 빌드 저장소 · 태그 |
|---|---|---|---|---|
| mpv (`Libmpv`) — **MPVKit 패치 적용** | v0.41.0 | LGPL-2.1+ (`-Dgpl=false`) | https://github.com/mpv-player/mpv/tree/v0.41.0 | mpvkit/MPVKit `1.1.0-n9.0.2` |
| FFmpeg (`Libavcodec` · `Libavdevice` · `Libavfilter` · `Libavformat` · `Libavutil` · `Libswresample` · `Libswscale`) | n9.0.2 | LGPL-3.0+ (version3 빌드) | https://github.com/FFmpeg/FFmpeg/tree/n9.0.2 | mpvkit/MPVKit `1.1.0-n9.0.2` |
| libplacebo (`Libplacebo`) | v7.360.1 | LGPL-2.1+ | https://code.videolan.org/videolan/libplacebo (미러 https://github.com/haasn/libplacebo) | mpvkit/libplacebo-build `7.360.1` |
| GnuTLS (`gnutls`) | 3.8.11 | LGPL-2.1+ | https://gitlab.com/gnutls/gnutls (미러 https://github.com/gnutls/gnutls) | mpvkit/gnutls-build `3.8.11` |
| Nettle (`nettle` · `hogweed`) | nettle_3.10_release_20240616 | LGPL-3.0+ 또는 GPL-2.0+ | https://git.lysator.liu.se/nettle/nettle | mpvkit/gnutls-build `3.8.11` |
| GMP (`gmp`) | v6.2.1 | LGPL-3.0+ 또는 GPL-2.0+ | https://gmplib.org (빌드에 쓴 미러 https://github.com/alisw/GMP) | mpvkit/gnutls-build `3.8.11` |
| GNU FriBidi (`Libfribidi`) | v1.0.16 | LGPL-2.1+ | https://github.com/fribidi/fribidi | mpvkit/libass-build `0.17.5` |
| libbluray (`Libbluray`) | 1.4.0 | LGPL-2.1+ | https://code.videolan.org/videolan/libbluray | mpvkit/libbluray-build `1.4.0` |
| uchardet (`Libuchardet`) | v0.0.8 | MPL-1.1 / GPL-2.0+ / LGPL-2.1+ | https://gitlab.freedesktop.org/uchardet/uchardet | mpvkit/libuchardet-build `0.0.8` |
| libass (`Libass`) | 0.17.5 | ISC | https://github.com/libass/libass | mpvkit/libass-build `0.17.5` |
| FreeType (`Libfreetype`) | VER-2-14-3 | FTL (또는 GPL-2.0) | https://github.com/freetype/freetype | mpvkit/libass-build `0.17.5` |
| HarfBuzz (`Libharfbuzz`) | 14.2.0 | MIT | https://github.com/harfbuzz/harfbuzz | mpvkit/libass-build `0.17.5` |
| libunibreak (`Libunibreak`) | libunibreak_6_1 | zlib | https://github.com/adah1972/libunibreak | mpvkit/libass-build `0.17.5` |
| Little CMS (`lcms2`) | lcms2.17 | MIT | https://github.com/mm2/Little-CMS | mpvkit/lcms2-build `2.17.0` |
| dav1d (`Libdav1d`) | 1.5.3 | BSD-2-Clause | https://code.videolan.org/videolan/dav1d | mpvkit/libdav1d-build `1.5.3` |
| libdovi (`Libdovi`) | libdovi-3.3.2 | MIT | https://github.com/quietvoid/dovi_tool | mpvkit/libdovi-build `3.3.2` |
| uavs3d (`Libuavs3d`) | 1.0 | BSD-3-Clause | https://github.com/uavs3/uavs3d | mpvkit/libuavs3d-build `1.2.1-fix` |
| MoltenVK (`libMoltenVK.a`) | v1.4.2 | Apache-2.0 | https://github.com/KhronosGroup/MoltenVK | mpvkit/moltenvk-build `1.4.2` |
| shaderc (`Libshaderc_combined`) | v2025.5 | Apache-2.0 (glslang · SPIRV-Tools 포함) | https://github.com/google/shaderc | mpvkit/libshaderc-build `2025.5.0` |
| OpenSSL (`Libssl` · `Libcrypto`) | 3.3.5 | Apache-2.0 | https://github.com/openssl/openssl | mpvkit/openssl-build `3.3.5` |

빌드 저장소 주소: `https://github.com/mpvkit/<저장소>/tree/<태그>`. 미리 빌드된 xcframework 는 각 저장소의 같은 태그 릴리스(`*.xcframework.zip`)에 있습니다.

### TVVLCKit — 동적 프레임워크

TVVLCKit **3.7.3**(LGPL-2.1+, CocoaPods). 배포 묶음 `TVVLCKit-3.7.3-319ed2c0-79128878.tar.xz`
(https://download.videolan.org/cocoapods/prod/TVVLCKit-3.7.3-319ed2c0-79128878.tar.xz, sha256 `b5f90c226ed54d9dc1c03901c60dc7749b74a53caace2c3047e4c0b7a063e46c`).
소스: VLCKit https://code.videolan.org/videolan/VLCKit (태그 3.7.3, 묶음 이름의 커밋 `319ed2c0`), VLC https://code.videolan.org/videolan/vlc (3.0 계열, 묶음 이름의 커밋 `79128878`).
앱은 수정하지 않은 바이너리를 `Frameworks/TVVLCKit.framework` 로 싣습니다.

### AMSMB2 · libsmb2 — 동적 프레임워크

AMSMB2 **4.0.3**(LGPL-2.1, https://github.com/amosavian/AMSMB2/tree/4.0.3, 커밋 `1726aaaf7adf63d7d1d2a0c5d1b0e635028215c0`).
libsmb2(LGPL-2.1+)를 하위 모듈로 소스에서 빌드해 함께 넣습니다: https://github.com/sahlberg/libsmb2 커밋 `aff9fa6ba9f41cfd3c15d184554601ec3f6d8d03`(`libsmb2-6.2-110-gaff9fa6`).
SwiftPM 이 소스에서 동적 프레임워크로 빌드하며 수정하지 않았습니다. `Frameworks/AMSMB2.framework` 로 실립니다.

빌드마다 정확한 판은 묶음의 `VERSIONS.txt` 가 기준입니다.

## 서면 제의 (Written offer)

NAS Player 의 각 빌드를 마지막으로 배포한 날로부터 **최소 3년 동안**, 요청하는 누구에게나 그 빌드에 쓰인 LGPL 라이브러리
(위 표의 LGPL 구성 요소, TVVLCKit/VLC, AMSMB2/libsmb2)의 **완전한 대응 소스 코드**와 이 저장소의 재링크 자료를,
배포에 드는 실비를 넘지 않는 비용(내려받기 링크는 무료)으로 제공합니다.
요청은 이 저장소의 [GitHub 이슈](https://github.com/yjinchoi/nasplayer-relink/issues/new)로 해 주세요(빌드 번호를 적어 주세요).

## 라이선스 전문

- [LICENSES/LGPL-2.1.txt](LICENSES/LGPL-2.1.txt) — GNU Lesser General Public License v2.1
- [LICENSES/LGPL-3.0.txt](LICENSES/LGPL-3.0.txt) — GNU Lesser General Public License v3.0
- [LICENSES/GPL-3.0.txt](LICENSES/GPL-3.0.txt) — GNU General Public License v3.0 (LGPL-3.0 이 참조)

이 저장소의 오브젝트 파일 · 스크립트는 LGPL 이 요구하는 범위에서 **자기 용도의 수정과, 그 수정을 디버깅하기 위한 리버스 엔지니어링**을 허용합니다.
각 라이브러리는 각자의 라이선스를 따릅니다.

---

## English

This repository provides the materials required by LGPL-2.1 §6 and LGPL-3.0 §4 to **relink the Apple TV app "NAS Player"** (App Store, bundle ID `com.nasplayer.tv`) against modified versions of the LGPL libraries it uses.
The app's own source code is not published; instead each App Store build has a release `build-<N>` containing the app's object files, the exact final link command, the unmodified static libraries, and the app bundle without its executable.

**Relinking**: install the Xcode version listed in `VERSIONS.txt` → replace one file in `libs/` (static: mpv, FFmpeg, libplacebo, GnuTLS, …) or `app/NASPlayer.app/Frameworks/` (dynamic: TVVLCKit, AMSMB2) with your modified build → `./link.sh` → `./sign.sh "<your signing identity>" <your tvOS provisioning profile>` → install on your own Apple TV with `xcrun devicectl device install app` or Xcode. A paid Apple Developer Program membership may be required to install on Apple TV.

**Sources**: see the tables above — MPVKit `1.1.0-n9.0.2` (mpv v0.41.0 with MPVKit's 3 patches, FFmpeg n9.0.2 LGPL-3 build, libplacebo v7.360.1, GnuTLS 3.8.11, Nettle 3.10, GMP 6.2.1, FriBidi 1.0.16, libbluray 1.4.0, uchardet 0.0.8, …), TVVLCKit 3.7.3, AMSMB2 4.0.3 with libsmb2 `aff9fa6b`.

**Written offer**: for at least three years after the last distribution of each build, we will provide anyone the complete corresponding source code of the LGPL libraries used in that build, together with these relinking materials, for no more than the cost of physically performing the distribution (download links are free). Please request it by opening an issue in this repository.

License texts: [LGPL-2.1](LICENSES/LGPL-2.1.txt), [LGPL-3.0](LICENSES/LGPL-3.0.txt), [GPL-3.0](LICENSES/GPL-3.0.txt).
