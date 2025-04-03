# Hướng dẫn cài đặt và triển khai dự án Seismic

Đây là hướng dẫn chi tiết để cài đặt các công cụ cần thiết, cấu hình môi trường, và triển khai một contract đơn giản trên nền tảng Seismic.

---

## 1. Cài đặt các gói phụ thuộc
Cập nhật hệ thống và cài đặt các công cụ cơ bản (nếu cần):

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```
## 2. Cấu hình Git
Bạn cần thiết lập thông tin Git trước khi bắt đầu. Có hai cách:
### Cách 1: Cấu hình toàn cục (khuyến nghị)
Thiết lập thông tin Git cho tất cả dự án trên máy:
```bash
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```
Thay you@example.com bằng email của bạn (ví dụ: nesa@gmail.com).
Thay Your Name bằng tên bạn muốn dùng (ví dụ: Nesa).
Ví dụ:
```bash
git config --global user.email "nesa@gmail.com"
git config --global user.name "Nesa"
```
### Cách 2: Cấu hình cục bộ (cho dự án này)
Nếu chỉ muốn cấu hình cho dự án hiện tại:
```bash
cd /home/nesa/my-seismic-project
git config user.email "you@example.com"
git config user.name "Your Name"
```
## Kiểm tra cấu hình
Xác nhận thông tin đã được thiết lập:
```bash
git config --global user.email  # Xem email
git config --global user.name   # Xem tên
```
## 3. Cài đặt Rust và Cargo
Nếu chưa có Rust và Cargo, cài đặt bằng lệnh sau:
```bash
curl https://sh.rustup.rs -sSf | sh
```
## 4. Cài đặt sfoundryup
Tải và thực thi script cài đặt sfoundryup:
```bash
curl -L \
     -H "Accept: application/vnd.github.v3.raw" \
     "https://api.github.com/repos/SeismicSystems/seismic-foundry/contents/sfoundryup/install?ref=seismic" | bash
source ~/.zshenv  # Hoặc ~/.bashrc hoặc ~/.zshrc tùy shell bạn dùng
```
Cài đặt sforge, sanvil, ssolc (quá trình này mất 5-20 phút tùy máy):
```bash
sfoundryup
source ~/.bashrc
```
## 5. Chạy node cục bộ
Mở terminal và chạy node Seismic cục bộ:
```bash
sanvil
```
## 6. Cài đặt Node.js
Cài đặt Node.js qua nvm (Node Version Manager):
```bash
# Tải và cài đặt nvm:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash
\. "$HOME/.nvm/nvm.sh"
# Cài đặt Node.js phiên bản 22:
nvm install 22
# Kiểm tra phiên bản:
node -v    # Nên hiển thị "v22.14.0"
nvm current # Nên hiển thị "v22.14.0"
npm -v     # Nên hiển thị "10.9.2"
```
Tải Node.js thủ công tại: nodejs.org.
## 7. Thiết lập dự án và deploy contract
### Bước 1: Tạo thư mục dự án
```bash
mkdir my-seismic-project
cd my-seismic-project
```
### Bước 2: Khởi tạo dự án với sforge
```bash
sforge init
sforge init --force  # Nếu cần ghi đè
```
### Bước 3: Viết contract đơn giản
Tạo cấu trúc thư mục:
```bash
mkdir -p packages/contracts/src packages/contracts/script
cd packages/contracts
```
Viết contract (src/Walnut.sol):
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract Walnut {
    uint256 public shellStrength;
    uint256 public kernelValue;

    constructor(uint256 _shellStrength, uint256 _kernelValue) {
        shellStrength = _shellStrength;
        kernelValue = _kernelValue;
    }
}
```
Viết script deploy (script/Walnut.s.sol):
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script, console} from "forge-std/Script.sol";
import {Walnut} from "../src/Walnut.sol";

contract WalnutScript is Script {
    Walnut public walnut;

    function run() public {
        uint256 deployerPrivateKey = vm.envUint("PRIVKEY");
        vm.startBroadcast(deployerPrivateKey);
        walnut = new Walnut(3, uint256(0));
        vm.stopBroadcast();
    }
}
```
### Bước 4: Thiết lập file .env
Tạo file .env:
```bash
nano .env
```
Dán nội dung sau vào .env:
```bash
RPC_URL=http://127.0.0.1:8545
PRIVKEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```
### Bước 5: Deploy contract
```bash
source .env
sforge script script/Walnut.s.sol:WalnutScript --rpc-url $RPC_URL --broadcast
```
###  Hoàn thành
# CHÚC CÁC BẠN THÀNH CÔNG #
