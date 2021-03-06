---
title: Assets APIs for digital asset management in Adobe Experience Manager as a Cloud Service 
description: Assets APIs allow for basic create-read-update-delete (CRUD) operations to manage assets, including binary, metadata, renditions, comments, and Content Fragments.
contentOwner: AG
---

# Assets as a Cloud Service APIs {#assets-cloud-service-apis}

<!-- 
Give a list of and overview of all reference information available.
* New upload method
* Javadocs
* Assets HTTP API documented at [https://helpx.adobe.com/experience-manager/6-5/assets/using/mac-api-assets.html](https://helpx.adobe.com/experience-manager/6-5/assets/using/mac-api-assets.html)

-->

## Asset upload {#asset-upload-technical}

Experience Manager as a cloud service provides a new way of uploading assets to the repository - direct binary upload to binary cloud storage. This section provides its technical overview.

### Overview of direct binary upload {#overview-binary-upload}

The high-level algorithm to upload a binary is:

1. Submit an HTTP request informing AEM of the intent to upload a new binary.
1. POST the contents of the binary to one or more URIs provided by the initiation request.
1. Submit an HTTP request to inform the server that the contents of the binary were successfully uploaded.

![Overview of direct binary upload protocol](assets/add-assets-technical.png)

Important differences compared to earlier versions of AEM include:

* Binaries do not go through AEM, which is now simply coordinating the upload process with the binary cloud storage configured for the deployment
* Binary cloud storage is fronted by a Content Delivery Network (CDN, Edge Network), which brings the upload endpoint closer to the client, thus helping to improve upload performance and user experience, especially for distributed teams uploading assets

This approach should provide more scalable and performant handling of asset uploads.

>[!NOTE]
>
>To review client code that implements this approach, refer to the open-source [aem-upload library](https://github.com/adobe/aem-upload)

### Initiate upload {#initiate-upload}

The first step is to submit an HTTP POST request to the folder where the asset should be created or updated; include the selector `.initiateUpload.json` to indicate that the request is to begin a binary upload. For example, the path to the folder where the asset should be created is `/assets/folder`. The POST request is `POST https://[aem_server]:[port]/content/dam/assets/folder.initiateUpload.json`.

The content type of the request body should be `application/x-www-form-urlencoded` form data, containing the following fields:

* `(string) fileName`: Required. The name of the asset as it will appear in the instance.
* `(number) fileSize`: Required. The total length, in bytes, of the binary to be uploaded.

A single request can be used to initiate uploads for multiple binaries, as long as each binary contains the required fields. If successful, the request responds with a `201` status code and a body containing JSON data in the following format:

```json
{
    "completeURI": "(string)",
    "folderPath": (string)",
    "files": [
        {
            "fileName": "(string)",
            "mimeType": "(string)",
            "uploadToken": "(string)",
            "uploadURIs": [
                "(string)"
            ]
        }
    ]
}
```

* `completeURI` (string): Invoke this URI when the binary finishes uploading. The URI can be an absolute or relative URI, and clients should be able to handle either. That is, the value can be `"https://author.acme.com/content/dam.completeUpload.json"` or `"/content/dam.completeUpload.json"` See [complete upload](#complete-upload).
* `folderPath` (string): Full path to the folder where the binary is being uploaded.
* `(files)` (array): A list of elements whose length and order will match the length and order of the list of binary information provided in the initiate request.
* `fileName` (string): The name of the corresponding binary, as supplied in the initiate request. This value should be included in the complete request.
* `mimeType` (string): The mime type of the corresponding binary, as supplied in the in initiate request. This value should be included in the complete request.
* `uploadToken` (string): An upload token for the corresponding binary. This value should be included in the complete request.
* `uploadURIs` (array): A list of strings whose values are full URIs to which the binary's content should be uploaded (see [Upload binary](#upload-binary)).
* `minPartSize` (number): The minimum length, in bytes, of data that may be provided to any one of the uploadURIs, if there is more than one URI.
* `maxPartSize` (number): The maximum length, in bytes, of data that may be provided to any one of the uploadURIs, if there is more than one URI.

### Upload binary {#upload-binary}

The output of initiating an upload will include one or more upload URI values. If more than one URI is provided, it's the client's responsibility to "split" the binary into parts and POST each part, in order, to each URI, in order. All URIs must be used, and each part must be larger than the minimum size and smaller than the maximum size as specified in the initiate response. Those requests will be fronted by CDN edge nodes to accelerate the upload of binaries.

One potential way of accomplishing this is to calculate the part size based on the number of upload URIs provided by the API. Example assuming the total size of the binary is 20,000 bytes and the number of upload URIs is 2:

* Calculate part size by dividing total size by number of URIs: 20,000 / 2 = 10,000
* POST byte range 0-9,999 of the binary to the first URI in the list of upload URIs
* POST byte range 10,000 - 19,999 of the binary to the second URI in the list of upload URIs

If successful, the server responds to each request with a `201` status code.

### Complete upload {#complete-upload}

After all the parts of a binary file are uploaded, submit an HTTP POST request to the complete URI provided by the initiation data. The content type of the request body should be `application/x-www-form-urlencoded` form data, containing the following fields.

|Fields | Type | Required or not | Description |
|---|---|---|---|
| `fileName` | String | Required | The name of the asset, as was provided by the initiation data. |
| `mimeType` | String | Required | The HTTP content type of the binary, as was provided by the initiation data. |
| `uploadToken` | String | Required | Upload token for the binary, as was provided by the initiation data. |
| `createVersion` | Boolean | Optional | If `True` and an asset with the specified name already exists, then Experience Manager creates a new version of the asset. |
| `versionLabel` | String | Optional | If a new version is created, the label associated with the new version of an asset . |
| `versionComment` | String | Optional | If a new version is created, the comments associated with the version. |
| `replace` | Boolean | Optional | If `True` and an asset with the specified name already exists, Experience Manager deletes the asset then re-create it. |

>![NOTE]
>
>If the asset already exists and neither `createVersion` nor `replace` is specified, then Experience Manager updates the asset's current version with the new binary.

Like the initiate process, the complete request data may contain information for more than one file.

The process of uploading a binary is not done until the complete URL is invoked for the file. Even if a file's binary is uploaded in its entirety, the asset will not be processed by the instance until its upload process is completed.

If successful, the server responds with a `200` status code.

### Open-source upload library {#open-source-upload-library}

To learn more about the upload algorithms or to build your own upload scripts and tools, Adobe provides open-source libraries and tools as a starting points:

* [Open-source aem-upload library](https://github.com/adobe/aem-upload)
* [Open-source command-line tool](https://github.com/adobe/aio-cli-plugin-aem)

### Deprecated asset upload APIs {#deprecated-asset-upload-api}

<!-- #ENGCHECK review / update the list of deprecated APIs below. -->

For Adobe Experience Manager as a Cloud Service only the new upload APIs are supported. The APIs from Adobe Experience Manager 6.5 are deprecated. The methods related to upload or update assets or renditions (any binary upload) are deprecated in the following APIs:

* [AEM Assets HTTP API](mac-api-assets.md)
* `AssetManager` Java API, like `AssetManager.createAsset(..)`

>[!MORELIKETHIS]
>
>* [Open-source aem-upload library](https://github.com/adobe/aem-upload).
>* [Open-source command-line tool](https://github.com/adobe/aio-cli-plugin-aem).

## Asset processing and post-processing workflows {#post-processing-workflows}

In Experience Manager, the asset processing is based on **[!UICONTROL Processing Profiles]** configuration that uses [asset microservices](asset-microservices-configure-and-use.md#get-started-using-asset-microservices). Processing does not require developer extensions.

For post-processing workflow configuration, use the standard workflows with extensions with custom steps.

## Support of workflow steps in post-processing workflow {#post-processing-workflows-steps}

Customers upgrading to Experience Manager as a Cloud Service from previous versions of Experience Manager can use asset microservices for processing of assets. The cloud-native asset microservices are much simpler to configure and use. A few workflow steps used in the [!UICONTROL DAM Update Asset] workflow in the previous version are not supported.

The following workflow steps are supported in Experience Manager as a Cloud Service.

* `com.day.cq.dam.similaritysearch.internal.workflow.process.AutoTagAssetProcess`
* `com.day.cq.dam.core.impl.process.CreateAssetLanguageCopyProcess`
* `com.day.cq.wcm.workflow.process.CreateVersionProcess`
* `com.day.cq.dam.similaritysearch.internal.workflow.smarttags.StartTrainingProcess`
* `com.day.cq.dam.similaritysearch.internal.workflow.smarttags.TransferTrainingDataProcess`
* `com.day.cq.dam.core.impl.process.TranslateAssetLanguageCopyProcess`
* `com.day.cq.dam.core.impl.process.UpdateAssetLanguageCopyProcess`
* `com.adobe.cq.workflow.replication.impl.ReplicationWorkflowProcess`
* `com.day.cq.dam.core.impl.process.DamUpdateAssetWorkflowCompletedProcess`

The following technical workflow models are either replaced by asset microservices or the support is not available.

* `com.day.cq.dam.core.impl.process.DamMetadataWritebackWorkflowCompletedProcess`
* `com.day.cq.dam.core.process.DeleteImagePreviewProcess`
* `com.day.cq.dam.s7dam.common.process.DMEncodeVideoWorkflowCompletedProcess`
* `com.day.cq.dam.core.process.GateKeeperProcess`
* `com.day.cq.dam.core.process.AssetOffloadingProcess`
* `com.day.cq.dam.core.process.MetadataProcessorProcess`
* `com.day.cq.dam.core.process.XMPWritebackProcess`
* `com.adobe.cq.dam.dm.process.workflow.DMImageProcess`
* `com.day.cq.dam.s7dam.common.process.S7VideoThumbnailProcess`
* `com.day.cq.dam.scene7.impl.process.Scene7UploadProcess`
* `com.day.cq.dam.s7dam.common.process.VideoProxyServiceProcess`
* `com.day.cq.dam.s7dam.common.process.VideoThumbnailDownloadProcess`
* `com.day.cq.dam.s7dam.common.process.VideoUserUploadedThumbnailProcess`
* `com.day.cq.dam.core.process.CreatePdfPreviewProcess`
* `com.day.cq.dam.core.process.CreateWebEnabledImageProcess`
* `com.day.cq.dam.video.FFMpegThumbnailProcess`
* `com.day.cq.dam.core.process.ThumbnailProcess`
* `com.day.cq.dam.cameraraw.process.CameraRawHandlingProcess`
* `com.day.cq.dam.core.process.CommandLineProcess`
* `com.day.cq.dam.pdfrasterizer.process.PdfRasterizerHandlingProcess`
* `com.day.cq.dam.core.process.AddPropertyWorkflowProcess`
* `com.day.cq.dam.core.process.CreateSubAssetsProcess`
* `com.day.cq.dam.core.process.DownloadAssetProcess`
* `com.day.cq.dam.word.process.ExtractImagesProcess`
* `com.day.cq.dam.word.process.ExtractPlainProcess`
* `com.day.cq.dam.video.FFMpegTranscodeProcess`
* `com.day.cq.dam.ids.impl.process.IDSJobProcess`
* `com.day.cq.dam.indd.process.INDDMediaExtractProcess`
* `com.day.cq.dam.indd.process.INDDPageExtractProcess`
* `com.day.cq.dam.core.impl.lightbox.LightboxUpdateAssetProcess`
* `com.day.cq.dam.pim.impl.sourcing.upload.process.ProductAssetsUploadProcess`
* `com.day.cq.dam.core.process.ScheduledPublishBPProcess`
* `com.day.cq.dam.core.process.ScheduledUnPublishBPProcess`
* `com.day.cq.dam.core.process.SendDownloadAssetEmailProcess`
* `com.day.cq.dam.core.impl.process.SendTransientWorkflowCompletedEmailProcess`

<!-- PPTX source: slide in add-assets.md - overview of direct binary upload section of 
https://adobe-my.sharepoint.com/personal/gklebus_adobe_com/_layouts/15/guestaccess.aspx?guestaccesstoken=jexDC5ZnepXSt6dTPciH66TzckS1BPEfdaZuSgHugL8%3D&docid=2_1ec37f0bd4cc74354b4f481cd420e07fc&rev=1&e=CdgElS
-->
