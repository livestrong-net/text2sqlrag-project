# Comprehensive System Dependencies for Unstructured Library on AWS Lambda (Amazon Linux 2023)

**Date:** 2026-01-23
**Context:** Running Unstructured.io library with PDF support on AWS Lambda using Amazon Linux 2023 base image
**Current Error:** `libGL.so.1: cannot open shared object file`
**Base Image:** `public.ecr.aws/lambda/python:3.12` (Amazon Linux 2023)

---

## Executive Summary

The Unstructured library requires extensive system dependencies for PDF processing, including Poppler, Tesseract, and OpenGL libraries. When deploying to AWS Lambda using the `public.ecr.aws/lambda/python:3.12` base image (Amazon Linux 2023), you must install these dependencies using the `dnf` package manager.

**The critical missing dependency causing the current error is `mesa-libGL`, which provides `libGL.so.1` for headless rendering operations.**

---

## 1. Understanding the Unstructured Library Dependencies

### Core System Requirements

According to [Unstructured documentation](https://docs.unstructured.io/open-source/introduction/overview), the library requires:

1. **Poppler** - For PDF processing (provides pdfinfo, pdftotext, pdftoppm, etc.)
2. **Tesseract** - For OCR capabilities in images and PDFs
3. **LibMagic** - For file type detection
4. **OpenGL Libraries** - For rendering operations (mesa-libGL)

### Internal Dependencies

- Unstructured uses `pdf2image` Python package internally
- `pdf2image` relies on Poppler's command-line tools (particularly `pdfinfo`)
- Document processing may trigger OpenGL operations requiring `libGL.so.1`

**Reference:** [Unstructured PDF Processing Documentation](https://deepwiki.com/Unstructured-IO/unstructured/4.1-pdf-and-image-processing)

---

## 2. Amazon Linux 2023 Package System

### Key Differences from Amazon Linux 2

- **Package Manager:** Uses `dnf` (not `yum`)
- **Minimal Container:** AL2023 Lambda images have limited packages pre-installed
- **Package Names:** Different from Debian/Ubuntu (e.g., `mesa-libGL` not `libgl1`)

**Reference:** [AWS Lambda Python 3.12 Runtime](https://aws.amazon.com/blogs/compute/python-3-12-runtime-now-available-in-aws-lambda/)

### Available Mesa Packages

According to [Amazon Linux 2023 documentation](https://docs.aws.amazon.com/linux/al2023/release-notes/all-packages.html), available packages include:

- `mesa-libGL` - Mesa runtime libraries (provides libGL.so.1)
- `mesa-libGL-devel` - Development files
- `mesa-libGLU` - Mesa utility library
- `mesa-libGLU-devel` - GLU development files
- `mesa-libgbm` - Generic Buffer Management
- `libglvnd-*` - Vendor-neutral GL dispatch library

---

## 3. Complete System Dependency List for AWS Lambda

### Recommended DNF Packages for Amazon Linux 2023

Based on research from multiple sources including [AWS Lambda base images issue #215](https://github.com/aws/aws-lambda-base-images/issues/215) and [AL2023 OpenCV setup](https://qiita.com/mychaelstyle/items/eef63a1249c66c4ff2bf), here's the comprehensive list:

```bash
# Core PDF processing
poppler
poppler-utils
poppler-glib

# OCR support
tesseract
tesseract-langpack-eng

# File type detection
file-libs

# OpenGL/Graphics libraries (CRITICAL for fixing libGL.so.1 error)
mesa-libGL
mesa-libGLU
mesa-libgbm
libglvnd-glx
libglvnd-egl

# X11 libraries (required by mesa-libGL)
libX11
libXext
libXrender
libxcb
libXau
libXdamage
libXfixes
libXxf86vm

# Additional graphics support
libpng
libtiff
libjpeg-turbo
openjpeg2

# System utilities
glib2
libmagic
fontconfig
```

### Debian/Ubuntu Equivalents (for reference)

If using a Debian-based image instead of AL2023, the equivalent packages are:

```bash
apt-get install -y \
    libmagic1 \
    poppler-utils \
    tesseract-ocr \
    libgl1-mesa-glx \
    libglib2.0-0
```

**Reference:** [Troubleshooting libGL.so.1 Errors](https://www.opensourcescripts.com/troubleshooting-importerror-libgl-so-1-in-python-applications/)

---

## 4. Understanding the libGL.so.1 Dependency Chain

### Why libGL.so.1 is Needed

Even in headless environments, PDF processing libraries may invoke rendering operations that require OpenGL:

1. **PDF rendering** - Converting PDF pages to images
2. **Image manipulation** - Resizing, rotating, format conversion
3. **Font rendering** - Text extraction with proper formatting

### Mesa on Headless Servers

From [Fedora documentation](https://fedoraproject.org/wiki/Changes/Vendor_Neutral_libGL):

- Modern systems use **libglvnd** (vendor-neutral GL dispatch)
- `mesa-libGL` provides the Mesa backend for libglvnd
- X11 client libraries are still required as dependencies
- Software rendering works without GPU in containerized environments

**Note:** While this adds overhead (~50-100MB), it's necessary for document processing operations.

---

## 5. Unstructured Docker Image Analysis

### Official Unstructured Dockerfile

The official [Unstructured Dockerfile](https://github.com/Unstructured-IO/unstructured/blob/main/Dockerfile) uses **Wolfi Linux** (apk package manager) and installs:

```dockerfile
# System packages (apk format - Wolfi Linux)
apk add:
  - mesa-gl
  - mesa-libgallium
  - libmagic
  - poppler
  - poppler-utils
  - poppler-glib
  - tesseract
  - libreoffice
  - pandoc
  - openjpeg
  - fontconfig
  - font-ubuntu
```

**Key Insight:** Even the official image includes full Mesa GL support (`mesa-gl`, `mesa-libgallium`)

**Reference:** [Unstructured Docker Documentation](https://docs.unstructured.io/open-source/installation/docker-installation)

---

## 6. Recommended Solutions

### Option A: Add Missing Packages to Current Dockerfile (Recommended)

Update your Lambda Dockerfile to install mesa-libGL and dependencies:

```dockerfile
FROM public.ecr.aws/lambda/python:3.12

# Install system dependencies for Unstructured library
RUN dnf install -y \
    # PDF processing
    poppler \
    poppler-utils \
    poppler-glib \
    # OCR
    tesseract \
    tesseract-langpack-eng \
    # File detection
    file-libs \
    # OpenGL/Graphics (FIXES libGL.so.1 ERROR)
    mesa-libGL \
    mesa-libGLU \
    mesa-libgbm \
    libglvnd-glx \
    # X11 dependencies
    libX11 \
    libXext \
    libXrender \
    libxcb \
    # Image libraries
    libpng \
    libjpeg-turbo \
    openjpeg2 \
    # System
    glib2 \
    fontconfig \
    && dnf clean all \
    && rm -rf /var/cache/dnf

# Rest of your Dockerfile...
```

**Estimated Size Impact:** +80-120MB to final image

### Option B: Use Debian-Based Python Image

As suggested in [AWS Lambda base images issue #215](https://github.com/aws/aws-lambda-base-images/issues/215), use a standard Python image:

```dockerfile
FROM python:3.12-slim

# Install AWS Lambda Runtime Interface Client
RUN pip install awslambdaric

# Install system dependencies (apt-get)
RUN apt-get update && apt-get install -y \
    libmagic1 \
    poppler-utils \
    tesseract-ocr \
    libgl1-mesa-glx \
    libglib2.0-0 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Rest of your Lambda setup...
```

**Pros:**
- All packages readily available
- Better documented
- Simpler package names

**Cons:**
- Larger base image
- Not the "official" Lambda runtime

### Option C: Use opencv-python-headless (Partial Solution)

If your use case doesn't require full OpenGL, consider using headless Python packages:

```python
# In requirements.txt
opencv-python-headless  # Instead of opencv-python
```

**Reference:** [Resolving libGL.so.1 Errors](https://itsmycode.com/importerror-libgl-so-1-cannot-open-shared-object-file-no-such-file-or-directory/)

**Note:** This only helps if OpenCV is the source of libGL dependency. Unstructured may still require it for PDF rendering.

---

## 7. Verification Commands

After building your Docker image, verify installations:

```bash
# Test if libGL.so.1 is available
docker run --rm your-image:tag ls -la /usr/lib64/libGL.so.1

# Check mesa-libGL installation
docker run --rm your-image:tag rpm -q mesa-libGL

# List all mesa packages
docker run --rm your-image:tag rpm -qa | grep mesa

# Test Unstructured import
docker run --rm your-image:tag python -c "from unstructured.partition.auto import partition; print('Success')"

# Test PDF processing
docker run --rm your-image:tag pdfinfo --version
docker run --rm your-image:tag tesseract --version
```

---

## 8. Common Issues and Troubleshooting

### Issue: Package Not Found

```
Error: Unable to find a match: mesa-libGL
```

**Solution:** Update dnf cache first:
```dockerfile
RUN dnf update -y && dnf install -y mesa-libGL
```

### Issue: Dependency Conflicts

**Solution:** Install packages in groups:
```dockerfile
# First install mesa
RUN dnf install -y mesa-libGL mesa-libGLU
# Then install poppler
RUN dnf install -y poppler poppler-utils
# Finally install tesseract
RUN dnf install -y tesseract
```

### Issue: Still Getting libGL.so.1 Error

**Debug Steps:**
1. Check if file exists: `find /usr -name "libGL.so.1"`
2. Check library path: `echo $LD_LIBRARY_PATH`
3. Add explicit path if needed:
   ```dockerfile
   ENV LD_LIBRARY_PATH=/usr/lib64:$LD_LIBRARY_PATH
   ```

**Reference:** [ModernGL Headless Setup](https://moderngl.readthedocs.io/en/5.6.2/the_guide/headless_ubunut18_server.html)

---

## 9. Additional Resources

### Official Documentation
- [Unstructured.io Documentation](https://docs.unstructured.io/open-source/introduction/overview)
- [AWS Lambda Container Images](https://docs.aws.amazon.com/lambda/latest/dg/python-image.html)
- [Amazon Linux 2023 Package List](https://docs.aws.amazon.com/linux/al2023/release-notes/all-packages.html)

### Community Resources
- [Using Poppler on AWS Lambda](https://www.petewilcock.com/using-poppler-pdftotext-and-other-custom-binaries-on-aws-lambda/)
- [PDF to Images in Lambda with Docker](https://medium.com/@bloggeraj392/converting-pdfs-to-images-in-aws-lambda-using-custom-containers-8d8d0e729beb)
- [Lambda Poppler Layer (Alternative approach)](https://github.com/jeylabs/aws-lambda-poppler-layer)

### GitHub Issues
- [AWS Lambda Base Images #215](https://github.com/aws/aws-lambda-base-images/issues/215) - Installing tesseract, poppler, libGL on AL2023
- [Unstructured GitHub](https://github.com/Unstructured-IO/unstructured) - Official repository

---

## 10. Recommended Next Steps

1. **Immediate Fix:** Add `mesa-libGL` and X11 dependencies to your Dockerfile
2. **Test:** Build and deploy the updated image
3. **Monitor:** Check CloudWatch logs for any remaining errors
4. **Optimize:** Remove any unnecessary packages to reduce image size
5. **Document:** Update your project README with final dependency list

---

## Appendix A: Minimal vs Full Installation

### Minimal (for basic PDF text extraction)
```bash
dnf install -y poppler-utils tesseract file-libs mesa-libGL libX11
```
**Size:** ~50-70MB

### Full (for complete Unstructured functionality)
```bash
dnf install -y poppler poppler-utils poppler-glib tesseract \
  tesseract-langpack-eng file-libs mesa-libGL mesa-libGLU mesa-libgbm \
  libglvnd-glx libX11 libXext libXrender libxcb libpng libjpeg-turbo \
  openjpeg2 glib2 fontconfig
```
**Size:** ~100-150MB

### Production (with cleanup)
```bash
dnf install -y poppler poppler-utils tesseract mesa-libGL mesa-libGLU \
  libglvnd-glx libX11 libXext glib2 && \
  dnf clean all && \
  rm -rf /var/cache/dnf
```
**Size:** ~80-120MB

---

## Appendix B: Package Manager Command Reference

### Query Package Dependencies
```bash
# Find what provides libGL.so.1
dnf provides libGL.so.1

# List package dependencies
dnf repoquery --requires mesa-libGL

# Check if package is installed
rpm -q mesa-libGL

# List all files in a package
rpm -ql mesa-libGL
```

### Debugging Installation Issues
```bash
# Search for packages
dnf search mesa

# Get package info
dnf info mesa-libGL

# List available versions
dnf list available mesa-libGL
```

---

## Appendix C: Comparison with Current Dockerfile

### Current Dockerfile (Debian-based)
Your current Dockerfile uses Debian packages:
```dockerfile
FROM python:3.12-slim as base

RUN apt-get update && apt-get install -y --no-install-recommends \
    libmagic1 \
    poppler-utils \
    tesseract-ocr \
    gcc \
    g++ \
    libspatialindex-dev \
    libgl1 \
    libglib2.0-0 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

### Equivalent for Lambda (Amazon Linux 2023)
```dockerfile
FROM public.ecr.aws/lambda/python:3.12

RUN dnf install -y \
    file-libs \
    poppler-utils \
    tesseract \
    gcc \
    gcc-c++ \
    spatial-index \
    mesa-libGL \
    glib2 \
    && dnf clean all \
    && rm -rf /var/cache/dnf
```

**Key Differences:**
- `apt-get` → `dnf`
- `libmagic1` → `file-libs`
- `tesseract-ocr` → `tesseract`
- `g++` → `gcc-c++`
- `libspatialindex-dev` → `spatial-index`
- `libgl1` → `mesa-libGL`
- `libglib2.0-0` → `glib2`

---

## Sources

1. [Unstructured.io Documentation - Overview](https://docs.unstructured.io/open-source/introduction/overview)
2. [Unstructured GitHub Repository](https://github.com/Unstructured-IO/unstructured)
3. [Unstructured PDF and Image Processing](https://deepwiki.com/Unstructured-IO/unstructured/4.1-pdf-and-image-processing)
4. [AWS Lambda Python 3.12 Runtime Announcement](https://aws.amazon.com/blogs/compute/python-3-12-runtime-now-available-in-aws-lambda/)
5. [AWS Lambda Base Images Issue #215](https://github.com/aws/aws-lambda-base-images/issues/215)
6. [Amazon Linux 2023 OpenCV Setup](https://qiita.com/mychaelstyle/items/eef63a1249c66c4ff2bf)
7. [Fedora Wiki - Vendor Neutral libGL](https://fedoraproject.org/wiki/Changes/Vendor_Neutral_libGL)
8. [Unstructured Docker Installation](https://docs.unstructured.io/open-source/installation/docker-installation)
9. [Using Poppler on AWS Lambda](https://www.petewilcock.com/using-poppler-pdftotext-and-other-custom-binaries-on-aws-lambda/)
10. [PDF to Images in AWS Lambda with Docker](https://medium.com/@bloggeraj392/converting-pdfs-to-images-in-aws-lambda-using-custom-containers-8d8d0e729beb)
11. [Troubleshooting libGL.so.1 Errors](https://www.opensourcescripts.com/troubleshooting-importerror-libgl-so-1-in-python-applications/)
12. [Resolving libGL.so.1 ImportError](https://itsmycode.com/importerror-libgl-so-1-cannot-open-shared-object-file-no-such-file-or-directory/)
13. [ModernGL Headless Setup](https://moderngl.readthedocs.io/en/5.6.2/the_guide/headless_ubunut18_server.html)
14. [Amazon Linux 2023 All Packages](https://docs.aws.amazon.com/linux/al2023/release-notes/all-packages.html)
15. [AWS Lambda Container Images Documentation](https://docs.aws.amazon.com/lambda/latest/dg/python-image.html)

---

**End of Document**

**Last Updated:** 2026-01-23
**Status:** Ready for implementation
**Recommended Action:** Update Dockerfile with Option A (Add mesa-libGL packages)
