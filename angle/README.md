# ANGLE - ほぼネイティブなグラフィックレイヤーエンジン

ANGLEの目標は、OpenGL ES APIコールをそのプラットフォームで利用可能なハードウェアサポートAPIのいずれかに変換することにより、複数のオペレーティングシステムのユーザーがWebGLおよびその他のOpenGL ESコンテンツをシームレスに実行できるようにすることです。ANGLEは現在、OpenGL ES 2.0、3.0、3.1からVulkan、デスクトップOpenGL、OpenGL ES、Direct3D 9、Direct3D 11への変換を提供しています。将来的には、ES 3.2、MetalとMacOSへの翻訳、Chrome OS、Fuchsiaのサポートが予定されています。

### バックレンダラーによるOpenGL ESのサポートレベル

|                |  Direct3D 9   |  Direct3D 11     |   Desktop GL   |    GL ES      |    Vulkan     |    Metal      |
|----------------|:-------------:|:----------------:|:--------------:|:-------------:|:-------------:|:-------------:|
| OpenGL ES 2.0  |    complete   |    complete      |    complete    |    complete   |    complete   |    complete   |
| OpenGL ES 3.0  |               |    complete      |    complete    |    complete   |    complete   |  in progress  |
| OpenGL ES 3.1  |               | [incomplete](doc/ES31StatusOnD3D11.md) |    complete    |    complete   |    complete   |               |
| OpenGL ES 3.2  |               |                  |  in progress   |  in progress  |  in progress  |               |

### バックレンダラーによるプラットフォーム対応

|              |    Direct3D 9  |   Direct3D 11  |   Desktop GL  |    GL ES    |   Vulkan    |    Metal    |
|-------------:|:--------------:|:--------------:|:-------------:|:-----------:|:-----------:|:-----------:|
| Windows      |    complete    |    complete    |   complete    |   complete  |   complete  |             |
| Linux        |                |                |   complete    |             |   complete  |             |
| Mac OS X     |                |                |   complete    |             |             | in progress |
| iOS          |                |                |               |             |             | in progress |
| Chrome OS    |                |                |               |   complete  |   planned   |             |
| Android      |                |                |               |   complete  |   complete  |             |
| GGP (Stadia) |                |                |               |             |   complete  |             |
| Fuchsia      |                |                |               |             |   complete  |             |

ANGLE v1.0.772 was certified compliant by passing the OpenGL ES 2.0.3 conformance tests in October 2011.

ANGLE has received the following certifications with the Vulkan backend:
* OpenGL ES 2.0: ANGLE 2.1.0.d46e2fb1e341 (Nov, 2019)
* OpenGL ES 3.0: ANGLE 2.1.0.f18ff947360d (Feb, 2020)
* OpenGL ES 3.1: ANGLE 2.1.0.f5dace0f1e57 (Jul, 2020)

また、ANGLEはEGL1.5仕様の実装も提供しています。

ANGLEは、Windowsプラットフォーム上のGoogle ChromeとMozilla Firefoxの両方で、デフォルトのWebGLバックエンドとして使用されています。Chromeは、高速化されたCanvas2D実装やNative Clientサンドボックス環境など、Windows上のすべてのグラフィックスレンダリングにANGLEを使用しています。

ANGLE シェーダーコンパイラの一部は、複数のプラットフォームにまたがる WebGL の実装でシェーダー検証およびトランスレータとして使用されています。Mac OS X、Linux、およびブラウザのモバイル版で使用されています。1 つのシェーダバリデータを持つことで、ブラウザやプラットフォーム間で一貫した GLSL ES シェーダのセットを受け入れることができます。シェーダ・トランスレータは、シェーダを他のシェーディング言語に翻訳したり、ネイティブ・グラフィック・ドライバのバグや癖に対処するためにシェーダの修正をオプションで適用したりするために使用できます。トランスレータは、Desktop GLSL、Vulkan GLSL、Direct3D HLSL、およびネイティブGLES2プラットフォーム用のESSLもターゲットにしています。

## 情報源

ANGLEリポジトリはChromiumプロジェクトによってホストされており、[オンラインで閲覧](https://chromium.googlesource.com/angle/angle)することができます。

    git clone https://chromium.googlesource.com/angle/angle


## ビルド

[Devセットアップ説明書](doc/DevSetup.md)をご覧ください。

## 貢献

* Join our [Google group](https://groups.google.com/group/angleproject) to keep up to date.
* Join us on [Slack](https://chromium.slack.com) in the #angle channel. You can
  follow the instructions on the [Chromium developer page](https://www.chromium.org/developers/slack)
  for the steps to join the Slack channel. For Googlers, please follow the
  instructions on this [document](https://docs.google.com/document/d/1wWmRm-heDDBIkNJnureDiRO7kqcRouY2lSXlO6N2z6M/edit?usp=sharing)
  to use your google or chromium email to join the Slack channel.
* [File bugs](http://anglebug.com/new) in the [issue tracker](https://bugs.chromium.org/p/angleproject/issues/list) (preferably with an isolated test-case).
* [Choose an ANGLE branch](doc/ChoosingANGLEBranch.md) to track in your own project.


* Read ANGLE development [documentation](doc).
* Look at [pending](https://chromium-review.googlesource.com/q/project:angle/angle+status:open)
  and [merged](https://chromium-review.googlesource.com/q/project:angle/angle+status:merged) changes.
* Become a [code contributor](doc/ContributingCode.md).
* Use ANGLE's [coding standard](doc/CodingStandard.md).
* Learn how to [build ANGLE for Chromium development](doc/BuildingAngleForChromiumDevelopment.md).
* Get help on [debugging ANGLE](doc/DebuggingTips.md).
* Go through [ANGLE's orientation](doc/Orientation.md) and sift through [starter projects](https://bugs.chromium.org/p/angleproject/issues/list?q=Hotlist%3DStarterBug). If you decide to take on any task, write a comment so you can get in touch with us, and more importantly, set yourself as the "owner" of the bug. This avoids having multiple people accidentally working on the same issue.


* Read about WebGL on the [Khronos WebGL Wiki](http://khronos.org/webgl/wiki/Main_Page).
* Learn about the initial ANGLE implementation details in the [OpenGL Insights chapter on ANGLE](http://www.seas.upenn.edu/~pcozzi/OpenGLInsights/OpenGLInsights-ANGLE.pdf) (this is not the most up-to-date ANGLE implementation details, it is listed here for historical reference only) and this [ANGLE presentation](https://drive.google.com/file/d/0Bw29oYeC09QbbHoxNE5EUFh0RGs/view?usp=sharing&resourcekey=0-CNvGnQGgFSvbXgX--Y_Iyg).
* Learn about the past, present, and future of the ANGLE implementation in [this presentation](https://docs.google.com/presentation/d/1CucIsdGVDmdTWRUbg68IxLE5jXwCb2y1E9YVhQo0thg/pub?start=false&loop=false).
* Watch a [short presentation](https://youtu.be/QrIKdjmpmaA) on the Vulkan back-end.
* Track the [dEQP test conformance](doc/dEQP-Charts.md)
* Read design docs on the [Vulkan back-end](src/libANGLE/renderer/vulkan/README.md)
* Read about ANGLE's [testing infrastructure](infra/README.md)
* View information on ANGLE's [supported extensions](doc/ExtensionSupport.md)
* If you use ANGLE in your own project, we'd love to hear about it!
