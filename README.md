# Virtual Try-On API Integration Guide

## Overview
This guide details the integration of the Nymbo Virtual Try-On API into a Node.js backend application for use in a React frontend. The API enables virtual garment fitting on human images, leveraging advanced AI for an immersive try-on experience.

For complete API reference and documentation, visit the [Nymbo Virtual Try-On on Hugging Face](https://huggingface.co/spaces/Nymbo/Virtual-Try-On).

## Installation

```bash
# Install required dependencies
npm install @gradio/client express multer
```

## API Configuration

### Environment Variables
```env
PORT=3000
GRADIO_API_URL=Nymbo/Virtual-Try-On
```

## API Implementation

### Basic Express Server Setup

```javascript
const express = require('express');
const multer = require('multer');
const { client } = require('@gradio/client');

const app = express();
const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: 10 * 1024 * 1024 } // 10MB limit
});

// Configure multipart form handling
const uploadFields = upload.fields([
  { name: 'human', maxCount: 1 },
  { name: 'garment', maxCount: 1 }
]);
```

### API Endpoint Implementation

```javascript
app.post('/api/virtual-tryon', uploadFields, async (req, res) => {
  try {
    const gradioApp = await client("Nymbo/Virtual-Try-On");
    
    // Convert uploaded files to appropriate format
    const humanImage = new Blob([req.files['human'][0].buffer], { 
      type: req.files['human'][0].mimetype 
    });
    const garmentImage = new Blob([req.files['garment'][0].buffer], { 
      type: req.files['garment'][0].mimetype 
    });

    // Default parameters
    const params = {
      maskingMode: req.body.maskingMode || "auto",
      denoisingSteps: parseInt(req.body.denoisingSteps) || 3,
      seed: parseInt(req.body.seed) || 3,
      useAutoMask: req.body.useAutoMask === 'true',
      enhanceOutput: req.body.enhanceOutput === 'true',
    };

    const result = await gradioApp.predict("/tryon", [
      { background: humanImage, layers: [], composite: null },
      garmentImage,
      params.maskingMode,
      params.useAutoMask,
      params.enhanceOutput,
      params.denoisingSteps,
      params.seed
    ]);

    res.json({
      success: true,
      data: {
        outputImage: result.data[0],
        maskedImage: result.data[1]
      }
    });

  } catch (error) {
    console.error('Virtual Try-On Error:', error);
    res.status(500).json({
      success: false,
      error: 'Failed to process virtual try-on request'
    });
  }
});
```

## API Parameters

### Request Body (multipart/form-data)

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| human | File | Yes | Human image to try the garment on |
| garment | File | Yes | Garment image to be tried on |
| maskingMode | String | No | Masking mode instructions (default: "auto") |
| denoisingSteps | Number | No | Number of denoising steps (default: 3) |
| seed | Number | No | Seed for reproducibility (default: 3) |
| useAutoMask | Boolean | No | Whether to use auto-masking (default: true) |
| enhanceOutput | Boolean | No | Whether to enhance output quality (default: true) |

### Response Format

```javascript
{
  "success": true,
  "data": {
    "outputImage": "base64_encoded_image_string",
    "maskedImage": "base64_encoded_image_string"
  }
}
```

## Frontend Integration Example

```javascript
async function performVirtualTryOn(humanImageFile, garmentImageFile) {
  const formData = new FormData();
  formData.append('human', humanImageFile);
  formData.append('garment', garmentImageFile);
  formData.append('denoisingSteps', '3');
  formData.append('seed', '3');
  formData.append('useAutoMask', 'true');
  formData.append('enhanceOutput', 'true');

  try {
    const response = await fetch('http://your-api-url/api/virtual-tryon', {
      method: 'POST',
      body: formData
    });

    const result = await response.json();
    
    if (result.success) {
      // Handle successful response
      return result.data;
    } else {
      throw new Error(result.error);
    }
  } catch (error) {
    console.error('Virtual Try-On request failed:', error);
    throw error;
  }
}
```

## Error Handling

The API implements proper error handling with appropriate HTTP status codes:

- 400: Bad Request (missing or invalid parameters)
- 415: Unsupported Media Type (invalid file format)
- 500: Internal Server Error (processing failure)

## Notes and Limitations

1. Maximum file size is set to 10MB by default
2. Supported image formats: JPEG, PNG
3. The API processes one garment try-on at a time
4. Processing time may vary based on image size and complexity

## Security Considerations

1. Implement proper authentication/authorization
2. Validate file types and sizes
3. Sanitize user inputs
4. Use HTTPS in production
5. Consider rate limiting for production use

## Example Usage with React

```javascript
import React, { useState } from 'react';

function VirtualTryOn() {
  const [humanImage, setHumanImage] = useState(null);
  const [garmentImage, setGarmentImage] = useState(null);
  const [result, setResult] = useState(null);
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);

    try {
      const result = await performVirtualTryOn(humanImage, garmentImage);
      setResult(result);
    } catch (error) {
      console.error('Try-on failed:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      <form onSubmit={handleSubmit}>
        <input
          type="file"
          onChange={(e) => setHumanImage(e.target.files[0])}
          accept="image/*"
        />
        <input
          type="file"
          onChange={(e) => setGarmentImage(e.target.files[0])}
          accept="image/*"
        />
        <button type="submit" disabled={loading}>
          {loading ? 'Processing...' : 'Try On'}
        </button>
      </form>

      {result && (
        <div>
          <img src={result.outputImage} alt="Try-on result" />
          <img src={result.maskedImage} alt="Masked image" />
        </div>
      )}
    </div>
  );
}

export default VirtualTryOn;
```
