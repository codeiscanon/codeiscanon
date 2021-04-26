---
title: "Multipart Upload to S3 using AWS SDK for Java"
date: 2021-04-26T11:37:31+10:00
url: multipart-upload-to-s3-using-aws-sdk-for-java
---
![Graph showing performance of simple and multipart S3 upload types](s3-upload-multipart.gif)

Introduction
====================
In a previous post, I  had explored [uploading files to S3](/upload-to-s3-using-aws-sdk-for-java) using `putObject` and its limitations. As recommended by AWS for any files larger than 100MB we should use multipart upload. There are a couple of ways to achieve this. I'll start with the simplest approach.

S3 multipart upload
====================
As the name suggests we can use the SDK to upload our object in parts instead of one big request. 

The AWS APIs require a lot of redundant information to be sent with every request, so I wrote a small abstraction layer.

The abstraction layer allows bytes to be added as the data is being generated. When the size of the payload goes above 25MB (the minimum limit for S3 parts) we create a multipart request and upload it to S3. This means that we are only keeping a subset of the data in memory at any point in time. This limit is configurable and can be increased if the use case requires it, but should be a minimum of 25MB.

```java
Stream<SampleData> sampleDataStream = // Generate your data
ByteArrayOutputStream stream = new ByteArrayOutputStream();
multipartUploadHelper.start();
sampleDataStream.forEach(sampleData -> {
    stream.writeBytes(objectMapper.writeValueAsBytes(sampleData));
    multipartUploadHelper.partUpload(limit,stream);
});
multipartUploadHelper.complete(stream);
```

Using this abstraction layer it is a lot simpler to understand the high-level steps of multipart upload.

```java
public MultipartUploadHelper start() {
    isValidStart();
    CreateMultipartUploadResponse multipartUpload = s3Client
        .createMultipartUpload(CreateMultipartUploadRequest.builder()
            .bucket(bucket)
            .key(destinationKey)
            .build());
    abortRuleId = multipartUpload.abortRuleId();
    uploadId = multipartUpload.uploadId();
    return this;
  }
  ```

Let's look at the individual steps of the multipart upload next.
When we start the multipart upload process, AWS provides an id to identify this process for the next steps - `uploadId`. We also get an `abortRuleId` in case we decide to not finish this multipart upload, possibly due to an error in the following steps. Leaving a multipart upload incomplete does not automatically delete the parts that have been uploaded. They are also not visible in the S3 UI. This means incomplete multipart uploads actually cost money until they are aborted. These can be automatically deleted after a set time by creating an S3 lifecycle rule - _Delete expired delete markers or incomplete multipart uploads_.

```java
public void partUpload(long limit, ByteArrayOutputStream byteArrayOutputStream) {
    if (byteArrayOutputStream.size() > limit) {
      partUpload(byteArrayOutputStream);
    }
  }

  private void partUpload(ByteArrayOutputStream byteArrayOutputStream) {
    Integer partNumber = parts.size() + 1;
    UploadPartResponse uploadPartResponse = s3Client.uploadPart(UploadPartRequest.builder()
        .bucket(bucket)
        .key(destinationKey)
        .uploadId(uploadId)
        .partNumber(partNumber)
        .build(), RequestBody.fromBytes(byteArrayOutputStream.toByteArray()));
    parts.add(CompletedPart.builder()
        .partNumber(partNumber)
        .eTag(uploadPartResponse.eTag())
        .build());
    byteArrayOutputStream.reset();
  }
  ```

The next step is to upload the data in parts. This method can be in a loop where data is being written line by line or any other small chunks of bytes. The limit value defines the minimum byte size we wait for before considering it a valid part. Once a part upload request is formed, the output stream is cleared so that there is no overlap with the next part. We also track the part number and the ETag response for the multipart upload. We will need them in the next step. ETag is in most cases the MD5 Hash of the object, which in our case would be a single part object.

```java
public void complete(ByteArrayOutputStream byteArrayOutputStream) {
    partUpload(byteArrayOutputStream);
    s3Client.completeMultipartUpload(CompleteMultipartUploadRequest.builder()
        .uploadId(uploadId)
        .bucket(bucket)
        .key(destinationKey)
        .multipartUpload(CompletedMultipartUpload.builder()
            .parts(parts).build())
        .build());
    log.info("Multipart Upload complete with " + parts.size() + " parts");
  }
```

The last step is to complete the multipart upload. We usually have to send the remaining bytes of data, which is going to be lower than the limit (25MB in our case). There are no size restrictions on this step. This will be our last part. 

We also have to pass the list of part numbers and their corresponding ETag when we complete a multipart upload. This is when S3 stitches them on the server-side and makes the entire file available.

The entire Helper class looks like this.

> MultipartUploadHelper.java
```java
public class MultipartUploadHelper {

  private final S3Client s3Client;
  private final String bucket;
  private final String destinationKey;
  private String abortRuleId;
  private String uploadId;
  private final List<CompletedPart> parts = new ArrayList<>();

  public MultipartUploadHelper(S3Client s3Client, String bucket, String destinationKey) {
    this.s3Client = s3Client;
    this.bucket = bucket;
    this.destinationKey = destinationKey;
  }

  public MultipartUploadHelper start() {
    isValidStart();
    CreateMultipartUploadResponse multipartUpload = s3Client
        .createMultipartUpload(CreateMultipartUploadRequest.builder()
            .bucket(bucket)
            .key(destinationKey)
            .build());
    abortRuleId = multipartUpload.abortRuleId();
    uploadId = multipartUpload.uploadId();
    return this;
  }

  private Boolean isValidStart() {
    if (isNull(abortRuleId) && isNull(uploadId) && parts.isEmpty()) {
      return true;
    }
    throw new RuntimeException("Invalid Multipart Upload Start");
  }

  public void partUpload(long limit, ByteArrayOutputStream byteArrayOutputStream) {
    if (byteArrayOutputStream.size() > limit) {
      partUpload(byteArrayOutputStream);
    }
  }

  private void partUpload(ByteArrayOutputStream byteArrayOutputStream) {
    Integer partNumber = parts.size() + 1;
    UploadPartResponse uploadPartResponse = s3Client.uploadPart(UploadPartRequest.builder()
        .bucket(bucket)
        .key(destinationKey)
        .uploadId(uploadId)
        .partNumber(partNumber)
        .build(), RequestBody.fromBytes(byteArrayOutputStream.toByteArray()));
    parts.add(CompletedPart.builder()
        .partNumber(partNumber)
        .eTag(uploadPartResponse.eTag())
        .build());
    byteArrayOutputStream.reset();
  }

  public void complete(ByteArrayOutputStream byteArrayOutputStream) {
    partUpload(byteArrayOutputStream);
    s3Client.completeMultipartUpload(CompleteMultipartUploadRequest.builder()
        .uploadId(uploadId)
        .bucket(bucket)
        .key(destinationKey)
        .multipartUpload(CompletedMultipartUpload.builder()
            .parts(parts).build())
        .build());
    log.info("Multipart Upload complete with " + parts.size() + " parts");
  }
```

I successfully uploaded a 1GB file and could continue with larger files using Localstack but it was extremely slow. I deployed the application to an EC2(Amazon Elastic Compute Cloud) Instance and continued testing larger files there. While Localstack is great for validating your code works it does have limitations in performance.

S3 multipart upload with async
====================
One inefficiency of the multipart upload process is that the data upload is synchronous. We should be able to upload the different parts of the data concurrently.

```java
SampleData sampleData = generator.nextObject(SampleData.class);
Stream<SampleData> sampleDataStream = Stream.generate(() -> sampleData).limit(count);
```

This is assuming that the data generation is actually faster than the S3 Upload. Using a random object generator was not performant enough for this. So I switched to using the same object repeatedly.

```java
private void partUpload(ByteArrayOutputStream byteArrayOutputStream) {
  Integer partNumber = atomicPartNumber.incrementAndGet();
  CompletableFuture<UploadPartResponse> uploadPartResponseCompletableFuture = s3Client
      .uploadPart(UploadPartRequest.builder()
          .bucket(bucket)
          .key(destinationKey)
          .uploadId(uploadId)
          .partNumber(partNumber)
          .build(), AsyncRequestBody.fromBytes(byteArrayOutputStream.toByteArray()));

  CompletableFuture<Object> objectCompletableFuture = uploadPartResponseCompletableFuture
      .handleAsync((uploadPartResponse, throwable) -> {

        parts.add(CompletedPart.builder()
            .partNumber(partNumber)
            .eTag(uploadPartResponse.eTag())
            .build());
        return this;
      });
  completableFuturesOfPartUploads.add(objectCompletableFuture);
  byteArrayOutputStream.reset();
}
```

The part upload step had to be changed to use the async methods provided in the SDK. And we use an AtomicInteger to keep track of the number of parts. But the overall logic stays the same.

```java
public void complete(ByteArrayOutputStream byteArrayOutputStream)
    throws InterruptedException, ExecutionException {
  partUpload(byteArrayOutputStream);

  CompletableFuture<Void> voidCompletableFuture = CompletableFuture
      .allOf(completableFuturesOfPartUploads.toArray(CompletableFuture[]::new));
  voidCompletableFuture.get();

  List<CompletedPart> sorted = parts.stream()
.sorted(Comparator.comparing(CompletedPart::partNumber)).collect(Collectors.toList());

  s3Client.completeMultipartUpload(CompleteMultipartUploadRequest.builder()
      .uploadId(uploadId)
      .bucket(bucket)
      .key(destinationKey)
      .multipartUpload(CompletedMultipartUpload.builder()
          .parts(sorted).build())
      .build()).get();
  log.info("Multipart Upload complete with " + parts.size() + " parts");
}
```


The complete step has similar changes, and we had to wait for all the parts to be uploaded before actually calling the SDK's complete multipart method.

I was getting the following error before I sorted the parts and their corresponding ETag.

`software.amazon.awssdk.services.s3.model.S3Exception: The list of parts was not in ascending order. Parts must be ordered by part number. (Service: S3, Status Code: 400, Request ID: T2DZJHWQ69SKWS15, Extended Request ID:`

Because of the asynchronous nature of the parts being uploaded, it is possible for the part numbers to be out of order and AWS expects them to be in order. Sorting the parts solved this problem.

With these changes, the total time for data generation and upload drops significantly. On instances with more resources, we could increase the thread pool size and get faster times. However, for our comparison, we have a clear winner. These results are from uploading various sized objects using a [t3.medium](https://medium.com/r/?url=https%3A%2F%2Faws.amazon.com%2Fec2%2Finstance-types%2Ft3%2F) AWS instance.

![Graph showing performance of simple and multipart S3 upload types](s3-upload-multipart.gif)

| File Size |  Simple  |  File     |   Multipart  | Async    |
|:----------|:---------|:----------|:-------------|:---------|
| 100kb     | 78   ms  | 76    ms  |200    ms     |230    ms |
| 1mb       | 165  ms  | 140   ms  |260    ms     |270    ms |
| 10mb      | 658  ms  | 400   ms  |400    ms     |510    ms |
| 100mb     | 8.9  s   | 2     s   |1.9    s      |1.3    s  |
| 500mb     | OOM      | 9     s   |8.8    s      |4.6    s  |
| 1gb       | OOM      | 19    s   |17     s      |8      s  |
| 2gb       | OOM      | 46    s   |32     s      |18     s  |
| 5gb       | OOM      | Exception |1 m 19 s      |43     s  |
| 10gb      | OOM      | Exception |2 m 38 s      |1 m 25 s  |
| 50gb      | OOM      | Exception |12m 46 s      |6 m 43 s  |
| 100gb     | OOM      | Exception |28m 17 s      |13m 31 s  |

Beyond this point, the only way I could improve on the performance for individual uploads was to scale the EC2 instances vertically. I have chosen EC2 Instances with higher network capacities. So here I am going from 5 → 10 → 25 → 50 gigabit network. I could upload a 100GB file in less than 7mins. However, a more in-depth cost-benefit analysis needs to be done for real-world use cases as the bigger instances are significantly more expensive. For the larger instances, CPU and memory was barely being used, but this  was the smallest instance with a 50-gigabit network that was available on AWS `ap-southeast-2` (Sydney).

![Graph showing performance of multipart S3 upload on various instance Types](s3-upload-instance-types.gif)


|     Name     |  Memory   |  vCPUs   |  Network   | Cost hourly |
|:-------------|:----------|:---------|:-----------|:------------|
| t3.medium    | 4.0 GiB   | 2 vCPUs  | 5 Gigabit  | $0.0528     |
| c5.large     | 4.0 GiB   | 2 vCPUs  | 10 Gigabit | $0.1110     |
| i3en.xlarge  | 32.0 GiB  | 4 vCPUs  | 25 Gigabit | $0.5420     |
| i3en.12xlarge| 384.0 GiB | 48vCPUs  | 50 Gigabit | $6.5040     |


Conclusion
====================


For all use cases of uploading files larger than 100MB, single or multiple,
async multipart upload is by far the best approach in terms of efficiency and I would choose that by default. However, if the team is not familiar with async programming & AWS S3, then s3PutObject from a file is a good middle ground. 

For files that are guaranteed to never exceed 5MB s3putObject is slightly more efficient. However, the difference in performance is ~ 100ms. I would choose a single mechanism from above and use it for all sizes for simplicity.
I would choose a 5 or 10-gigabit network to run my application as the increase in speed does not justify the costs. However, this can be different in your AWS region.

I must highlight some caveats of the results -

* The exact values of requests per second might vary based on OS, hardware, load, and many other terms. These tests compare the performance of different methods and point to the ones that are noticeably faster than others.
* The processing by the example was minimal with default settings. If you add logic to your endpoints, data processing, database connections, and so on, your results will be different.

It was quite a fun experience to stretch this simple use case to its limits. Have you used S3 or any alternatives or have an interesting use case? Please share in the comments about your experience.