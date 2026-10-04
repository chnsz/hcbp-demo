# Introduction

## What is Video on Demand (VOD)

Video on Demand (VOD) is a one-stop video on-demand service provided by Huawei Cloud for scenarios such as audio and video websites, online education, e-commerce live streaming, and enterprise training. It delivers end-to-end capabilities from audio and video upload, transcoding, and storage to acceleration, distribution, and playback. You can quickly build a stable, smooth, and secure video on-demand service without building a complex audio and video processing and distribution system yourself.

VOD supports multiple upload methods, including console upload, API/SDK upload, and URL pulling upload, and supports media category management, audio and video transcoding, screenshot, watermark, encryption, CDN acceleration and distribution, and playback authentication. With media categories, you can group massive audio and video resources for easy retrieval and maintenance; with transcoding templates and workflows, you can transcode source files into playback formats suitable for different terminals and network conditions.

With VOD, enterprises can elastically scale audio and video processing and distribution capabilities on demand, reduce the O&M cost of self-built systems, and ensure smooth video playback and content security through CDN acceleration and playback authentication, laying a solid foundation for the continuous operation of online video services.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Video on Demand (VOD), helping you understand how to efficiently manage cloud VOD resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for VOD resources. These best practices will help you quickly get started with automated VOD deployment and lay a solid foundation for subsequent media asset, category, transcoding, and distribution management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Media Category and Media Asset](media_asset_with_category.md) - Introduces how to use Terraform to automatically deploy a media category and a media asset, including media category creation, media asset creation by URL pulling, and association between the media asset and the category.

## Reference Materials

- [Huawei Cloud Video on Demand Product Documentation](https://support.huaweicloud.com/vod/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
