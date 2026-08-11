---
title: "Kiểm thử hệ thống"
date: 2026-08-03
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

# KIỂM THỬ HỆ THỐNG

Sau khi hoàn tất quá trình triển khai hạ tầng và các dịch vụ AWS, nhóm tiến hành kiểm thử hệ thống **Live Auction** nhằm xác nhận khả năng hoạt động của các chức năng chính trong môi trường thực tế.

Quá trình kiểm thử tập trung vào xác thực và phân quyền tài khoản, REST API, nghiệp vụ quản lý phiên đấu giá, kết nối WebSocket, luồng đặt giá theo thời gian thực, tải hình ảnh lên Amazon S3, phân phối nội dung qua Amazon CloudFront và các cơ chế bảo mật của hệ thống.

## Nội dung kiểm thử

1. [Tổng quan và môi trường kiểm thử](5.5.1-Overview-and-Test-Environment/)
2. [Phương pháp và định dạng test case](5.5.2-Test-Methodology-and-Test-Case-Format/)
3. [Kiểm thử xác thực và phân quyền](5.5.3-Authentication-and-Authorization-Testing/)
4. [Kiểm thử REST API và nghiệp vụ quản lý đấu giá](5.5.4-REST-API-and-Auction-Management-Business-Logic-Testing/)
5. [Kiểm thử kết nối WebSocket và cập nhật thời gian thực](5.5.5-WebSocket-Connection-and-Real-Time-Update-Testing/)
6. [Kiểm thử luồng đặt giá đầu cuối](5.5.6-End-to-End-Bidding-Flow-Testing/)
7. [Kiểm thử S3, tải ảnh và CloudFront](5.5.7-S3-Image-Upload-and-CloudFront-Testing/)
8. [Kiểm thử bảo mật hệ thống](5.5.8-System-Security-Testing/)