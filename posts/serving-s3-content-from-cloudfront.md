## Serving S3 content from CloudFront

- make a private Bucket

- let CloudFront set the Bucket permissions

- in the Cloudfront behaviour settings set the `Allowed HTTP Methods` attribute to `GET, HEAD, OPTIONS` (emphasis on OPTIONS, because of CORS)

- in S3, make sure a valid CORS policy is set:

```
<CORSConfiguration>
  <CORSRule>
    <AllowedOrigin>*</AllowedOrigin>
    <AllowedMethod>GET</AllowedMethod>
    <MaxAgeSeconds>3000</MaxAgeSeconds>
    <AllowedHeader>Authorization</AllowedHeader>
  </CORSRule>
</CORSConfiguration>
```

- go to S3 -> Bucket -> Permissions -> you'll need to explicitly click the *SAVE* button to set the default policy that's visible in there

- testing CORS:

```
curl https://test.example.com/image.png -H "Origin: website.example.com" -H "Access-Control-Request-Method: GET" -X OPTIONS
```

- this will return 403 if CORS is not properly set up or 200 if it was ok

- source: http://blog.celingest.com/en/2014/10/02/tutorial-using-cors-with-cloudfront-and-s3/

- Route53 -> A Record -> Alias -> enter the cloudfront URL (with a CNAME you just get a redirect!)

```
{
    "Version": "2012-10-17",
    "Id": "my-policy",
    "Statement": [
        {
            "Sid": "Allow read action from the CloudFront distribution",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity [CLOUDFRONT_ID]"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::pass-cdn.blacklane.com/*"
        }
    ]
}
```

```
        {
            "Sid": "Allow any action from within these VPCs",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::[BUCKET_NAME]",
                "arn:aws:s3:::[BUCKET_NAME]/*"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:sourceVpce": [
                        "vpce-[VPC_ID]"
                    ]
                }
            }
        },
        {
            "Sid": "Allow any action from these IPs",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::[BUCKET_NAME]",
                "arn:aws:s3:::[BUCKET_NAME]/*"
            ],
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": [
                        "[SOME_IP]/32"
                    ]
                }
            }
        }
```
