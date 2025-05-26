# S3 Image Compression Project

This project automatically compresses images uploaded to an S3 bucket using AWS Lambda. When an image is uploaded to the `input/` folder, it triggers a Lambda function that compresses the image and saves it to the `output/` folder.

## Architecture Overview

```
S3 Bucket (input/) → S3 Event → Lambda Function → Compressed Image → S3 Bucket (output/)
```

## Features

- **Automatic Processing**: Images uploaded to `input/` folder are automatically processed
- **Format Support**: Supports JPG, JPEG, and PNG image formats
- **Configurable Compression**: Quality can be adjusted via environment variables
- **Error Handling**: Comprehensive error handling with CloudWatch logging
- **Cost Effective**: Serverless architecture with pay-per-use pricing

## Prerequisites

- AWS Account with appropriate permissions
- Basic understanding of AWS Console navigation

## Setup Instructions

### 1. Create S3 Bucket

1. Navigate to **S3** in the AWS Console
2. Click **Create bucket**
3. Enter bucket name (e.g., `your-image-compression-bucket`)
4. Choose your preferred region
5. Leave other settings as default and click **Create bucket**
6. Open the created bucket
7. Click **Create folder** and create a folder named `input/`
8. Click **Create folder** again and create a folder named `output/`

### 2. Create IAM Role

1. Navigate to **IAM** in the AWS Console
2. Click **Roles** in the left sidebar
3. Click **Create role**
4. Select **AWS service** as trusted entity type
5. Select **Lambda** as the service
6. Click **Next**
7. Search for and select the following policies:
   - `AWSLambdaBasicExecutionRole`
   - `AmazonS3FullAccess`
   - `CloudWatchLogsFullAccess`
8. Click **Next**
9. Enter role name: `ImageCompressionLambdaRole`
10. Click **Create role**

### 3. Pillow Layer Setup

Since PIL/Pillow is not available in the default Lambda runtime, you need to add it as a layer.

#### Option 1: Use Pre-built Public Layer (Recommended)

Use this public ARN for the Pillow layer (replace `us-east-1` with your region):
```
arn:aws:lambda:us-east-1:770693421928:layer:Klayers-p39-pillow:1
```

#### Option 2: Create Custom Layer

1. Navigate to **Lambda** in the AWS Console
2. Click **Layers** in the left sidebar
3. Click **Create layer**
4. Enter layer name: `pillow-layer`
5. Upload a ZIP file containing the Pillow library (you'll need to create this locally)
6. Select compatible runtimes: Python 3.9, 3.10, 3.11
7. Click **Create**

### 4. Create Lambda Function

1. Navigate to **Lambda** in the AWS Console
2. Click **Create function**
3. Select **Author from scratch**
4. Enter function name: `ImageCompressionFunction`
5. Select runtime: **Python 3.9**
6. Under **Permissions**, select **Use an existing role**
7. Choose the role: `ImageCompressionLambdaRole`
8. Click **Create function**

### 5. Add Pillow Layer to Lambda Function

1. In your Lambda function, scroll down to the **Layers** section
2. Click **Add a layer**
3. Select **Specify an ARN**
4. Enter the Pillow layer ARN: `arn:aws:lambda:us-east-1:770693421928:layer:Klayers-p39-pillow:1` (adjust region as needed)
5. Click **Add**

### 6. Configure Lambda Function

1. In the **Code** tab, replace the default code with the Lambda function code provided below
2. Go to the **Configuration** tab
3. Click **Environment variables** in the left panel
4. Click **Edit** and add the following variables:
   - Key: `OUTPUT_PREFIX`, Value: `output/`
   - Key: `QUALITY`, Value: `75`
5. Click **Save**
6. In **General configuration**, set:
   - Timeout: 1 minute
   - Memory: 512 MB

### 7. Configure S3 Event Trigger

1. Go back to your S3 bucket
2. Click on the **Properties** tab
3. Scroll down to **Event notifications**
4. Click **Create event notification**
5. Enter event name: `ImageUploadTrigger`
6. Select **All object create events**
7. In **Prefix**, enter: `input/`
8. In **Destination**, select **Lambda function**
9. Choose your Lambda function: `ImageCompressionFunction`
10. Click **Save changes**

## Lambda Function Code

Here's the complete Lambda function code:

```python
import boto3
import os
from PIL import Image
import io
import urllib.parse

# Initialize the S3 client
s3 = boto3.client('s3')

# Get environment variables for output prefix and compression quality
output_prefix = os.environ.get('OUTPUT_PREFIX', 'output/')
quality = int(os.environ.get('QUALITY', 75))  # Compression quality (0-100)

def lambda_handler(event, context):
    # Process each file upload event individually
    for record in event['Records']:
        # Extract bucket name and object key (file path)
        bucket = record['s3']['bucket']['name']
        key = urllib.parse.unquote_plus(record['s3']['object']['key'])

        # Only process supported image types
        if not key.lower().endswith(('.jpg', '.jpeg', '.png')):
            print(f"Incorrect image format: {key}")
            return {
                'statusCode': 400,
                'body': 'Incorrect image format'
            }

        try:
            # Download the image file from S3
            response = s3.get_object(Bucket=bucket, Key=key)
            image_data = response['Body'].read()

            # Open the image using Pillow
            img = Image.open(io.BytesIO(image_data))

            # Prepare an in-memory buffer to hold the compressed image
            compressed_io = io.BytesIO()

            # Save the image to buffer with compression (JPEG format)
            img.save(compressed_io, format='JPEG', optimize=True, quality=quality)
            compressed_io.seek(0)  # Reset buffer pointer to the beginning

            # Build the output file key (path), changing folder and file name
            output_key = key.replace("input/", output_prefix, 1).rsplit('.', 1)[0] + "_compressed.jpg"

            # Upload the compressed image back to S3 in the output/ directory
            s3.put_object(
                Bucket=bucket,
                Key=output_key,
                Body=compressed_io,
                ContentType='image/jpeg'
            )

            print(f"Compressed image saved to: {output_key}")
            return {
                'statusCode': 200,
                'body': f'Successfully compressed and saved to {output_key}'
            }

        except Exception as e:
            print(f"Error processing image {key}: {str(e)}")
            return {
                'statusCode': 500,
                'body': f'Error processing image: {str(e)}'
            }
```

## IAM Role Permissions

The Lambda function requires the following permissions:

### Trust Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Attached Policies
- **AWSLambdaBasicExecutionRole**: Basic Lambda execution permissions
- **AmazonS3FullAccess**: Full S3 access for reading and writing objects
- **CloudWatchLogsFullAccess**: CloudWatch Logs access for logging

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `OUTPUT_PREFIX` | `output/` | S3 prefix for compressed images |
| `QUALITY` | `75` | JPEG compression quality (0-100) |

### Supported Formats

- **Input**: JPG, JPEG, PNG
- **Output**: JPEG (compressed)

## Usage

1. Upload an image to the `input/` folder in your S3 bucket
2. The Lambda function will automatically trigger
3. The compressed image will be saved to the `output/` folder with `_compressed.jpg` suffix

### Example

1. Go to your S3 bucket in the AWS Console
2. Navigate to the `input/` folder
3. Click **Upload** and select an image file (JPG, JPEG, or PNG)
4. Click **Upload**
5. Wait a few moments for processing
6. Check the `output/` folder for the compressed image with `_compressed.jpg` suffix

## Monitoring

Monitor the function execution through:

- **CloudWatch Logs**: `/aws/lambda/ImageCompressionFunction`
- **CloudWatch Metrics**: Lambda function metrics
- **S3 Events**: S3 bucket event notifications

## Troubleshooting

### Common Issues

1. **Permission Denied**: Ensure IAM role has proper S3 permissions
2. **Layer Issues**: Verify Pillow layer is correctly attached
3. **Timeout**: Increase Lambda timeout for large images
4. **Memory Issues**: Increase Lambda memory allocation

### Debugging

Check CloudWatch Logs for detailed error messages:

1. Navigate to **CloudWatch** in the AWS Console
2. Click **Log groups** in the left sidebar
3. Look for `/aws/lambda/ImageCompressionFunction`
4. Click on the log group to view execution logs

## Cost Optimization

- **Lambda**: Pay only for execution time
- **S3**: Standard storage pricing
- **Data Transfer**: Minimal costs for same-region transfers

## Security Considerations

- Use least-privilege IAM policies
- Enable S3 bucket versioning
- Consider encrypting S3 objects
- Monitor access patterns

## Future Enhancements

- Support for additional image formats (WebP, TIFF)
- Batch processing capabilities
- Image resizing options
- Advanced compression algorithms
- SNS notifications for processing status
