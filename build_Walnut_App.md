# Hướng Dẫn Chi Tiết Từng Bước Để Tạo Giao Diện Web Cơ Bản Cho Dự Án Walnut App Trên Seismic Devnet
---
## Chuẩn bị trước khi bắt đầu
### Trước khi bắt đầu, bạn cần chuẩn bị một số công cụ cơ bản:
- Máy tính với hệ điều hành hỗ trợ terminal (Windows, macOS, hoặc Linux).
- Đã cài đặt trình duyệt web (Chrome, Firefox, v.v.) để kiểm tra giao diện.
- Đã cài đặt ví MetaMask (hoặc ví tương thích) trên trình duyệt và đã nhận ETH từ faucet tại https://faucet-2.seismicdev.net/.

## Cài đặt các thành phần cần thiết
### 1. Cài đặt các công cụ cơ bản
Nếu bạn chưa cài đặt các công cụ cơ bản, hãy chạy lệnh sau (trên Linux, macOS có thể cần điều chỉnh):
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```
### 2. Cài đặt Node.js và npm
Đảm bảo bạn đã cài đặt Node.js (phiên bản >= 18.x) và npm:
```bash
# Cài đặt nvm (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash
# Kích hoạt nvm
\. "$HOME/.nvm/nvm.sh"
# Cài đặt Node.js phiên bản 23
nvm install 23
# Kiểm tra phiên bản Node.js
node -v  # Nên hiển thị "v23.11.0"
# Kiểm tra phiên bản npm
npm -v  # Nên hiển thị "10.9.2"
```
### 3. Cài đặt Bun
Bun là công cụ thay thế npm để quản lý dependencies. Cài đặt Bun bằng lệnh:
```bash
curl -fsSL https://bun.sh/install | bash
```
Kiểm tra phiên bản:
```bash
bun -v  # Nên hiển thị phiên bản mới nhất (ví dụ: 1.1.x)
```
### 4. Cài đặt Foundry
Foundry là công cụ để xây dựng và kiểm tra hợp đồng thông minh. Cài đặt bằng lệnh:
```bash
curl -L https://foundry.paradigm.xyz | bash
```
Sau đó chạy:
```bash
foundryup
```
Kiểm tra:
```bash
forge --version  # Nên hiển thị phiên bản mới nhất
```
### 5. Cài đặt Seismic Tools
Cài đặt sforge và sanvil từ Seismic Systems:
```bash
curl -L https://seismic.systems/install.sh | bash
```
Kiểm tra:
```bash
sforge --version  # Nên hiển thị phiên bản
sanvil --version  # Nên hiển thị phiên bản
```
---
## Bắt đầu
### 1. Tạo thư mục dự án và cấu trúc monorepo
Tạo thư mục dự án và cấu trúc monorepo:
```bash
mkdir walnut-app
cd walnut-app
mkdir -p packages/contracts packages/frontend
```
### 2. Khởi tạo monorepo tại thư mục gốc
Khởi tạo monorepo và cấu hình các file cơ bản:
```bash
bun init -y && rm index.ts && rm tsconfig.json && touch .prettierrc && touch .gitmodules
```
Sửa package.json tại thư mục gốc:
```json
{
  "workspaces": ["packages/**"],
  "dependencies": {},
  "devDependencies": {
    "@trivago/prettier-plugin-sort-imports": "^5.2.1",
    "prettier": "^3.4.2"
  }
}
```
Sửa .prettierrc:
```json
{
  "semi": false,
  "tabWidth": 2,
  "singleQuote": true,
  "printWidth": 80,
  "trailingComma": "es5",
  "plugins": ["@trivago/prettier-plugin-sort-imports"],
  "importOrder": [
    "<TYPES>^(?!@)([^.].*$)</TYPES>",
    "<TYPES>^@(.*)$</TYPES>",
    "<TYPES>^[./]</TYPES>",
    "^(?!@)([^.].*$)",
    "^@(.*)$",
    "^[./]"
  ],
  "importOrderParserPlugins": ["typescript", "jsx", "decorators-legacy"],
  "importOrderSeparation": true,
  "importOrderSortSpecifiers": true
}
```
Sửa .gitignore:
```
# Compiler files
cache/
out/
# Ignores development broadcast logs
!/broadcast
/broadcast/*/31337/
/broadcast/**/dry-run/
# Docs
docs/
# Dotenv file
.env
node_modules/
```
Sửa .gitmodules:
```
[submodule "packages/contracts/lib/forge-std"]
path = packages/contracts/lib/forge-std
url = https://github.com/foundry-rs/forge-std
```
### 3. Khởi tạo packages/contracts
Chuyển đến thư mục contracts và khởi tạo dự án Foundry:
```bash
cd packages/contracts
sforge init --no-commit && rm -rf .github
rm -f src/Counter.sol test/Counter.t.sol script/Counter.s.sol
touch src/Walnut.sol test/Walnut.t.sol script/Walnut.s.sol
```
Sửa .gitignore trong packages/contracts:
```
.env
broadcast/
cache/
```
Thêm Walnut.sol trong packages/contracts/src:
```solidity
// SPDX-License-Identifier: MIT License
pragma solidity ^0.8.13;

contract Walnut {
    uint256 initialShellStrength;
    uint256 shellStrength;
    uint256 round;
    uint256 initialKernel;
    uint256 kernel;
    mapping(uint256 => mapping(address => uint256)) hitsPerRound;

    event Hit(uint256 indexed round, address indexed hitter, uint256 remaining);
    event Shake(uint256 indexed round, address indexed shaker);
    event Reset(uint256 indexed newRound, uint256 shellStrength);

    constructor(uint256 _shellStrength, uint256 _kernel) {
        initialShellStrength = _shellStrength;
        shellStrength = _shellStrength;
        initialKernel = _kernel;
        kernel = _kernel;
        round = 1;
    }

    function getShellStrength() public view returns (uint256) {
        return shellStrength;
    }

    function getRound() public view returns (uint256) {
        return round;
    }

    function hit() public requireIntact {
        shellStrength--;
        hitsPerRound[round][msg.sender]++;
        emit Hit(round, msg.sender, shellStrength);
    }

    function shake(uint256 _numShakes) public requireIntact {
        kernel += _numShakes;
        emit Shake(round, msg.sender);
    }

    function reset() public requireCracked {
        shellStrength = initialShellStrength;
        kernel = initialKernel;
        round++;
        emit Reset(round, shellStrength);
    }

    function look(address contributor) public view requireCracked returns (uint256) {
        require(hitsPerRound[round][contributor] > 0, "NOT_A_CONTRIBUTOR");
        return kernel;
    }

    function set_number(uint256 _kernel) public {
        kernel = _kernel;
    }

    modifier requireCracked() {
        require(shellStrength == 0, "SHELL_INTACT");
        _;
    }

    modifier requireIntact() {
        require(shellStrength > 0, "SHELL_ALREADY_CRACKED");
        _;
    }
}
```
Thêm Walnut.s.sol trong packages/contracts/script:
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
        walnut = new Walnut(3, 0); // shellStrength = 3, kernel = 0
        vm.stopBroadcast();
    }
}
```
Thêm .env trong packages/contracts:
```
CHAIN_ID=5124
RPC_URL=https://node-2.seismicdev.net/rpc
PRIVKEY=<your-private-key>
```
Thay <your-private-key> bằng khóa riêng của ví bạn (đã nhận ETH từ faucet).

Triển khai hợp đồng:
```bash
source .env
sforge script script/Walnut.s.sol:WalnutScript --rpc-url $RPC_URL --broadcast
```
Ghi lại địa chỉ hợp đồng từ packages/contracts/broadcast/Walnut.s.sol/5124/run-latest.json.

### 4. Khởi tạo packages/frontend
Chuyển đến thư mục frontend và khởi tạo dự án React:
```bash
cd ../frontend
bun create vite@latest . -- --template react-ts
```
Cài đặt dependencies:
```bash
bun add viem react-toastify @heroicons/react
```
Sửa package.json trong packages/frontend:
```json
{
  "name": "walnut-frontend",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite --host",
    "build": "tsc -b && vite build",
    "lint": "eslint .",
    "preview": "vite preview"
  },
  "dependencies": {
    "@heroicons/react": "^2.1.5",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-toastify": "^10.0.6",
    "viem": "^2.22.3"
  },
  "devDependencies": {
    "@eslint/js": "^9.13.0",
    "@types/react": "^18.3.11",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.3",
    "eslint": "^9.13.0",
    "eslint-plugin-react-hooks": "^5.0.0",
    "eslint-plugin-react-refresh": "^0.4.12",
    "typescript": "^5.5.3",
    "vite": "^5.4.8"
  }
}
```
Tạo cấu trúc thư mục trong packages/frontend:
```bash
mkdir -p src/lib src/hooks src/components
touch src/lib/constants.ts src/lib/wallet.ts src/hooks/useContract.ts src/components/WalnutGame.tsx
```
Thêm .gitignore trong packages/frontend:
```
node_modules
dist
.env
```
Thêm constants.ts trong packages/frontend/src/lib:
```typescript
import { defineChain } from 'viem'

export const seismicDevnet5124 = defineChain({
  id: 5124,
  name: 'Seismic Devnet',
  network: 'seismic-devnet',
  nativeCurrency: {
    decimals: 18,
    name: 'Ether',
    symbol: 'ETH',
  },
  rpcUrls: {
    default: {
      http: ['https://node-2.seismicdev.net/rpc'],
      webSocket: ['wss://node-2.seismicdev.net/ws'],
    },
  },
  blockExplorers: {
    default: { name: 'Seismic Explorer', url: 'https://explorer-2.seismicdev.net' },
  },
  testnet: true,
})

export const CONTRACT_ADDRESS = '0xYourContractAddress' as `0x${string}` // Thay bằng địa chỉ hợp đồng đã triển khai

export const WALNUT_ABI = [
  {
    inputs: [
      { internalType: 'uint256', name: '_shellStrength', type: 'uint256' },
      { internalType: 'uint256', name: '_kernel', type: 'uint256' },
    ],
    stateMutability: 'nonpayable',
    type: 'constructor',
  },
  {
    anonymous: false,
    inputs: [
      { indexed: true, internalType: 'uint256', name: 'round', type: 'uint256' },
      { indexed: true, internalType: 'address', name: 'hitter', type: 'address' },
      { indexed: false, internalType: 'uint256', name: 'remaining', type: 'uint256' },
    ],
    name: 'Hit',
    type: 'event',
  },
  {
    anonymous: false,
    inputs: [
      { indexed: true, internalType: 'uint256', name: 'round', type: 'uint256' },
      { indexed: true, internalType: 'address', name: 'shaker', type: 'address' },
    ],
    name: 'Shake',
    type: 'event',
  },
  {
    anonymous: false,
    inputs: [
      { indexed: true, internalType: 'uint256', name: 'newRound', type: 'uint256' },
      { indexed: false, internalType: 'uint256', name: 'shellStrength', type: 'uint256' },
    ],
    name: 'Reset',
    type: 'event',
  },
  {
    inputs: [],
    name: 'getShellStrength',
    outputs: [{ internalType: 'uint256', name: '', type: 'uint256' }],
    stateMutability: 'view',
    type: 'function',
  },
  {
    inputs: [],
    name: 'getRound',
    outputs: [{ internalType: 'uint256', name: '', type: 'uint256' }],
    stateMutability: 'view',
    type: 'function',
  },
  {
    inputs: [],
    name: 'hit',
    outputs: [],
    stateMutability: 'nonpayable',
    type: 'function',
  },
  {
    inputs: [{ internalType: 'uint256', name: '_numShakes', type: 'uint256' }],
    name: 'shake',
    outputs: [],
    stateMutability: 'nonpayable',
    type: 'function',
  },
  {
    inputs: [],
    name: 'reset',
    outputs: [],
    stateMutability: 'nonpayable',
    type: 'function',
  },
  {
    inputs: [{ internalType: 'address', name: 'contributor', type: 'address' }],
    name: 'look',
    outputs: [{ internalType: 'uint256', name: '', type: 'uint256' }],
    stateMutability: 'view',
    type: 'function',
  },
  {
    inputs: [{ internalType: 'uint256', name: '_kernel', type: 'uint256' }],
    name: 'set_number',
    outputs: [],
    stateMutability: 'nonpayable',
    type: 'function',
  },
] as const
```
Thêm wallet.ts trong packages/frontend/src/lib:
```typescript

import { createPublicClient, createWalletClient, custom, http } from 'viem'
import { seismicDevnet5124 } from './constants'

export const connectWallet = async () => {
  if (!window.ethereum) {
    throw new Error('Please install MetaMask!')
  }

  const [account] = await window.ethereum.request({ method: 'eth_requestAccounts' })

  const publicClient = createPublicClient({
    chain: seismicDevnet5124,
    transport: http(),
  })

  const walletClient = createWalletClient({
    chain: seismicDevnet5124,
    transport: custom(window.ethereum),
  })

  return { account: account as `0x${string}`, publicClient, walletClient }
}
```
Thêm useContract.ts trong packages/frontend/src/hooks:
```typescript

import { useState, useEffect } from 'react'
import { PublicClient, WalletClient } from 'viem'
import { CONTRACT_ADDRESS, WALNUT_ABI } from '../lib/constants'

export const useContract = (publicClient: PublicClient | null, walletClient: WalletClient | null) => {
  const [shellStrength, setShellStrength] = useState<bigint | null>(null)
  const [round, setRound] = useState<bigint | null>(null)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    const fetchData = async () => {
      if (!publicClient) return

      try {
        const shellStrengthData = await publicClient.readContract({
          address: CONTRACT_ADDRESS,
          abi: WALNUT_ABI,
          functionName: 'getShellStrength',
        })
        setShellStrength(shellStrengthData as bigint)

        const roundData = await publicClient.readContract({
          address: CONTRACT_ADDRESS,
          abi: WALNUT_ABI,
          functionName: 'getRound',
        })
        setRound(roundData as bigint)
      } catch (err: any) {
        setError('Failed to fetch data: ' + err.message)
      }
    }

    fetchData()
    const interval = setInterval(fetchData, 5000)
    return () => clearInterval(interval)
  }, [publicClient])

  const hit = async () => {
    if (!walletClient || !publicClient) throw new Error('Wallet not connected')

    const { request } = await publicClient.simulateContract({
      account: walletClient.account,
      address: CONTRACT_ADDRESS,
      abi: WALNUT_ABI,
      functionName: 'hit',
    })

    const hash = await walletClient.writeContract(request)
    await publicClient.waitForTransactionReceipt({ hash })
  }

  const shake = async (numShakes: number) => {
    if (!walletClient || !publicClient) throw new Error('Wallet not connected')

    const { request } = await publicClient.simulateContract({
      account: walletClient.account,
      address: CONTRACT_ADDRESS,
      abi: WALNUT_ABI,
      functionName: 'shake',
      args: [numShakes],
    })

    const hash = await walletClient.writeContract(request)
    await publicClient.waitForTransactionReceipt({ hash })
  }

  const reset = async () => {
    if (!walletClient || !publicClient) throw new Error('Wallet not connected')

    const { request } = await publicClient.simulateContract({
      account: walletClient.account,
      address: CONTRACT_ADDRESS,
      abi: WALNUT_ABI,
      functionName: 'reset',
    })

    const hash = await walletClient.writeContract(request)
    await publicClient.waitForTransactionReceipt({ hash })
  }

  const look = async (address: `0x${string}`) => {
    if (!publicClient) throw new Error('Public client not connected')

    const result = await publicClient.readContract({
      address: CONTRACT_ADDRESS,
      abi: WALNUT_ABI,
      functionName: 'look',
      args: [address],
    })

    return result as bigint
  }

  return { shellStrength, round, hit, shake, reset, look, error }
}
```
Thêm WalnutGame.tsx trong packages/frontend/src/components:
```typescript

import { useState, useEffect } from 'react'
import { connectWallet } from '../lib/wallet'
import { useContract } from '../hooks/useContract'
import { PublicClient, WalletClient } from 'viem'
import { ToastContainer, toast } from 'react-toastify'
import 'react-toastify/dist/ReactToastify.css'
import { WalletIcon } from '@heroicons/react/24/solid'

// Import Google Fonts trực tiếp
const styles = `
  @import url('https://fonts.googleapis.com/css2?family=Bungee&display=swap');

  @keyframes bounce {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(-10px); }
  }

  .bounce {
    animation: bounce 2s infinite;
  }
`

export default function WalnutGame() {
  const [account, setAccount] = useState<`0x${string}` | null>(null)
  const [publicClient, setPublicClient] = useState<PublicClient | null>(null)
  const [walletClient, setWalletClient] = useState<WalletClient | null>(null)
  const [numShakes, setNumShakes] = useState<number>(1)
  const { shellStrength, round, hit, shake, reset, look, error } = useContract(publicClient, walletClient)

  const handleConnect = async () => {
    try {
      const { account, publicClient, walletClient } = await connectWallet()
      setAccount(account)
      setPublicClient(publicClient)
      setWalletClient(walletClient)
      toast.success('Wallet connected successfully!')
    } catch (err: any) {
      toast.error(err.message)
    }
  }

  const handleHit = async () => {
    try {
      await hit()
      toast.success('Hit successful! Shell strength decreased.')
    } catch (err: any) {
      toast.error(error || 'Failed to hit.')
    }
  }

  const handleShake = async () => {
    try {
      await shake(numShakes)
      toast.success(`Shook ${numShakes} times! Kernel increased.`)
    } catch (err: any) {
      toast.error(error || 'Failed to shake.')
    }
  }

  const handleReset = async () => {
    try {
      await reset()
      toast.success('Game reset! New round started.')
    } catch (err: any) {
      toast.error(error || 'Failed to reset.')
    }
  }

  const handleLook = async () => {
    if (!account) return
    const result = await look(account)
    if (result !== null) {
      toast.info(`Kernel: ${result}`)
    } else {
      toast.error(error || 'Failed to look.')
    }
  }

  return (
    <>
      <style>{styles}</style>
      <div className="min-h-screen bg-gradient-to-br from-amber-200 via-green-200 to-brown-300 flex items-center justify-center p-6">
        <div className="container mx-auto flex flex-col md:flex-row gap-8 max-w-5xl">
          {/* Phần chơi game - Bên trái */}
          <div className="w-full md:w-1/2 bg-white p-8 rounded-2xl shadow-xl border-4 border-green-300">
            <h1 className="text-6xl font-extrabold mb-6 text-center text-amber-600 font-['Bungee'] drop-shadow-[0_2px_2px_rgba(0,0,0,0.3)] bounce">
              Walnut Game
            </h1>

            {!account ? (
              <div className="flex justify-center">
                <button
                  onClick={handleConnect}
                  className="flex items-center space-x-3 bg-black text-white py-4 px-8 rounded-xl hover:bg-gray-800 hover:scale-105 transition-all duration-300 text-xl font-extrabold shadow-md border-2 border-white"
                >
                  <WalletIcon className="h-6 w-6" />
                  <span>Connect Wallet</span>
                </button>
              </div>
            ) : (
              <div>
                <p className="mb-3 text-base font-semibold text-green-700">
                  Connected: {account.slice(0, 6)}...{account.slice(-4)}
                </p>
                <p className="mb-3 text-xl font-medium text-brown-800">
                  Round: {round?.toString() || 'Loading...'}
                </p>
                <p className="mb-6 text-xl font-medium text-brown-800">
                  Shell Strength: {shellStrength?.toString() || 'Loading...'}
                </p>

                {error && <p className="text-red-600 mb-6 font-medium">{error}</p>}

                <div className="space-y-6">
                  <button
                    onClick={handleHit}
                    className="w-full bg-green-600 text-white py-4 rounded-xl hover:bg-green-700 hover:scale-105 transition-all duration-300 text-lg font-semibold shadow-md border-2 border-green-400"
                  >
                    Hit
                  </button>

                  <div>
                    <input
                      type="number"
                      value={numShakes}
                      onChange={(e) => setNumShakes(Number(e.target.value))}
                      className="w-full p-3 border-2 border-brown-300 rounded-xl mb-3 focus:outline-none focus:ring-2 focus:ring-green-500 text-lg bg-amber-50"
                      placeholder="Enter shakes"
                      min="1"
                    />
                    <button
                      onClick={handleShake}
                      className="w-full bg-amber-500 text-white py-4 rounded-xl hover:bg-amber-600 hover:scale-105 transition-all duration-300 text-lg font-semibold shadow-md border-2 border-amber-400"
                    >
                      Shake
                    </button>
                  </div>

                  <button
                    onClick={handleReset}
                    className="w-full bg-red-600 text-white py-4 rounded-xl hover:bg-red-700 hover:scale-105 transition-all duration-300 text-lg font-semibold shadow-md border-2 border-red-400"
                  >
                    Reset
                  </button>

                  <button
                    onClick={handleLook}
                    className="w-full bg-brown-600 text-white py-4 rounded-xl hover:bg-brown-700 hover:scale-105 transition-all duration-300 text-lg font-semibold shadow-md border-2 border-brown-400"
                  >
                    Look
                  </button>
                </div>
              </div>
            )}
          </div>

          {/* Phần hướng dẫn - Bên phải */}
          <div className="w-full md:w-1/2 bg-amber-50 p-8 rounded-2xl shadow-xl border-4 border-green-300">
            <h2 className="text-3xl font-semibold text-brown-800 mb-6 font-['Bungee']">How to Play</h2>
            <ol className="list-decimal list-inside text-brown-700 space-y-4">
              <li className="text-lg">Click "Connect Wallet" to start.</li>
              <li className="text-lg">Press "Reset" to begin a new round (Shell Strength = 3).</li>
              <li className="text-lg">Enter a number and press "Shake" to increase the kernel.</li>
              <li className="text-lg">Press "Hit" 3 times to break the shell (until Shell Strength = 0).</li>
              <li className="text-lg">Press "Look" to see your kernel reward!</li>
            </ol>
          </div>
        </div>
        <ToastContainer position="top-right" autoClose={3000} hideProgressBar={false} />
      </div>
    </>
  )
}
```
Sửa App.tsx trong packages/frontend/src:
```typescript

import WalnutGame from './components/WalnutGame'

function App() {
  return (
    <div>
      <WalnutGame />
    </div>
  )
}

export default App
```
### 5. Chạy giao diện web
Chuyển đến thư mục frontend và chạy ứng dụng:
```bash

cd packages/frontend
bun dev --host
```
Mở giao diện tại:
http://localhost:5173 (trên máy local).
http://192.168.1.4:5173 (trên mạng nội bộ, nếu cần).

### 6. Kết luận
Bạn đã hoàn thành việc thiết lập và triển khai dự án "Walnut App" trên Seismic Devnet với giao diện web cơ bản! Hướng dẫn này bao gồm toàn bộ quá trình:
- Cài đặt môi trường.
- Cấu hình monorepo.
- Viết và triển khai hợp đồng thông minh.
- Xây dựng giao diện web với React để chơi trò chơi.

Nếu bạn gặp bất kỳ vấn đề nào
Nếu bạn gặp bất kỳ vấn đề nào trong quá trình thực hiện (lỗi hiển thị, lỗi chức năng, hoặc cần cải thiện giao diện), hãy cho mình biết để mình hỗ trợ nhé! Chúc bạn thành công và vui vẻ với dự án!

