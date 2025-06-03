# 區塊鏈投票平台技術文件：開發者指南

## 快速入門

本文檔提供區塊鏈投票平台的技術實現指南，幫助開發者理解系統架構和關鍵技術點。

> 💡 **提示**：開始前請確保已安裝 Node.js (v14+)、npm/yarn 和 MetaMask 瀏覽器擴展。

### 環境準備

```bash
# 克隆代碼倉庫
git clone https://github.com/your-org/blockchain-voting-platform.git

# 安裝依賴
cd blockchain-voting-platform
npm install

# 啟動本地開發環境
npm run dev
```

## 核心技術組件

### 智能合約實現

投票平台的核心功能基於以太坊智能合約實現，主要包含兩個合約：

#### ElectionContract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract ElectionContract {
    // 選舉數據結構
    struct Election {
        string title;
        uint256 startTime;
        uint256 endTime;
        bool finalized;
        mapping(uint256 => Candidate) candidates;
        uint256 candidateCount;
    }
    
    // 候選人數據結構
    struct Candidate {
        string name;
        string info;
        uint256 voteCount;
    }
    
    // 主要功能實現
    function createElection(string memory _title, uint256 _duration) public { /* ... */ }
    function vote(uint256 _electionId, uint256 _candidateId) public { /* ... */ }
    function finalizeElection(uint256 _electionId) public { /* ... */ }
}
```

> ⚠️ **警告**：投票合約一旦部署就不可更改，請務必進行全面測試和安全審計。

#### ZK-Proof 集成

匿名投票利用零知識證明技術實現：

```javascript
// 前端 ZK-Proof 集成示例
async function generateZKProof(voterData, electionId) {
  const witness = await snarkjs.calculateWitness({
    voterKey: voterData.privateKey,
    electionId: electionId,
    merkleProof: voterData.merkleProof
  });
  
  const { proof, publicSignals } = await snarkjs.proof(witness);
  return { proof, publicSignals };
}
```

## 系統架構詳解

### 前後端交互流程

![系統架構](https://example.com/architecture-diagram.png)

1. **用戶認證流程**：
   - 使用 Firebase Auth 處理初始認證
   - 生成和驗證 Merkle 證明確認投票資格
   - 連接 Web3 錢包獲取區塊鏈身份

2. **投票流程**：
   ```
   用戶 -> 選擇候選人 -> 生成零知識證明 -> 調用智能合約 -> 提交投票 -> 獲取交易收據
   ```

## 關鍵技術實現指南

### Merkle 樹驗證機制

```javascript
// 後端生成 Merkle 樹示例
function generateMerkleTree(voterList) {
  const leaves = voterList.map(voter => 
    keccak256(Buffer.from(voter.address.slice(2), 'hex')));
  const tree = new MerkleTree(leaves, keccak256, { sortPairs: true });
  return tree;
}

// 智能合約中的驗證
function verifyVoter(bytes32[] memory _proof, bytes32 _root, bytes32 _leaf) 
  internal pure returns (bool) {
  return MerkleProof.verify(_proof, _root, _leaf);
}
```

### 前端 Web3 集成

使用 Ethers.js 與區塊鏈交互：

```javascript
// React 組件中的合約調用示例
const voteForCandidate = async (electionId, candidateId) => {
  try {
    setIsLoading(true);
    const provider = new ethers.providers.Web3Provider(window.ethereum);
    const signer = provider.getSigner();
    const contract = new ethers.Contract(
      CONTRACT_ADDRESS, 
      ElectionContractABI, 
      signer
    );
    
    // 生成零知識證明
    const { proof, publicSignals } = await generateZKProof(
      userData, 
      electionId
    );
    
    // 調用合約方法
    const tx = await contract.vote(electionId, candidateId, proof, publicSignals);
    await tx.wait();
    
    // 更新 UI
    setIsLoading(false);
    showSuccess("投票成功!");
  } catch (error) {
    setIsLoading(false);
    showError("投票失敗: " + error.message);
  }
};
```

## 測試與部署

### 合約測試

使用 Hardhat 進行合約測試：

```javascript
describe("Election Contract", function() {
  it("Should allow eligible voters to cast votes", async function() {
    // 部署測試合約
    const ElectionContract = await ethers.getContractFactory("ElectionContract");
    const election = await ElectionContract.deploy();
    await election.deployed();
    
    // 創建選舉
    await election.createElection("Test Election", 86400); // 1天
    
    // 投票測試
    await election.vote(0, 1); // 投給候選人 ID 1
    
    // 驗證結果
    const result = await election.getResults(0);
    expect(result[1].voteCount).to.equal(1);
  });
});
```

### 部署流程

```bash
# 編譯合約
npx hardhat compile

# 部署到測試網
npx hardhat run scripts/deploy.js --network goerli

# 驗證部署
npx hardhat verify --network goerli DEPLOYED_CONTRACT_ADDRESS
```

## 安全最佳實踐

1. **合約安全**
   - 使用 OpenZeppelin 合約庫
   - 實施重入攻擊防護
   - 設置適當的權限控制

2. **隱私保護**
   - 使用零知識證明確保投票隱私
   - 資料存儲加密
   - 鏈下保存敏感資訊

> ℹ️ **注意**：定期進行安全審計，特別是在添加新功能後。

## 故障排除

| 問題 | 可能原因 | 解決方案 |
|------|---------|---------|
| 合約調用失敗 | Gas 費用不足 | 增加 Gas 限制或降低合約複雜度 |
| MetaMask 無法連接 | 網絡不匹配 | 確保 MetaMask 設置為正確網絡 |
| ZK-Proof 生成失敗 | 參數錯誤 | 檢查輸入參數格式和完整性 |

更多技術問題，請參考完整文檔或提交 GitHub issue。