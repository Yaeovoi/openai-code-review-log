本次代码变更主要涉及 GitHub Actions 工作流的重构与整合，删除了本地构建和 Maven 构建的两个旧流程，保留并优化了基于远程 JAR 包的执行流程。以下是详细的评审意见：

### 1. 代码质量与可读性
*   **优点 - 结构精简**：删除了冗余的 `main-local.yml` 和 `main-maven-jar.yml`，统一使用远程 JAR 方式，降低了维护成本，逻辑更加清晰。
*   **优点 - 步骤合并**：将原本分散的 "Get repository name", "Get branch name" 等多个步骤合并为一个 "Get commit info" 步骤，显著提高了 YAML 文件的可读性，同时也减少了步骤间的上下文切换开销。
*   **建议 - 环境变量命名**：新增了 `COMMIT_REPO` 和 `COMMIT_SHA`，命名风格统一，但在 `Run Code Review` 步骤中，环境变量传递非常清晰，这是一个很好的改进。

### 2. 潜在的安全隐患
*   **高风险 - 远程 JAR 执行**：
    *   流程通过 `wget` 下载 JAR 包并直接执行，存在供应链攻击风险。如果源仓库被篡改或下载链接被劫持，CI 环境将执行恶意代码。
    *   **建议**：建议在下载后校验 JAR 包的 SHA256 或 GPG 签名，或者使用 GitHub Actions 提供的官方 Action（如 `actions/download-artifact` 或专门的 Release Action）来获取依赖，增加来源的可信度。
*   **中风险 - 版本固定与不可控**：
    *   使用 `releases/latest/download/...` 意味着每次构建都会使用最新版本的 SDK。虽然方便，但如果最新版 SDK 存在 Bug 或破坏性变更，会导致所有使用该工作流的仓库 CI 失败。
    *   **建议**：建议在配置中明确指定版本号（如 `download/V1.1/...`），以保证构建环境的稳定性与可复现性。
*   **配置清理**：删除了微信相关的 Secret 配置（`WEIXIN_*`），增加了飞书配置，确认 `secrets` 中的敏感信息已在 GitHub 仓库设置中同步更新，避免运行时报错。

### 3. 性能问题
*   **优化点 - 步骤合并**：将多个 Shell 脚本步骤合并为一个，减少了 Runner 启动子进程和文件系统 IO 的次数，对 CI 执行速度有微小但积极的提升。
*   **Action 版本升级**：`actions/checkout` 和 `actions/setup-java` 从 v2 升级到 v4，v4 版本通常包含性能优化和对 Node.js 运行时的更好支持，这是很好的优化。

### 4. 最佳实践建议
*   **JDK 发行版选择**：
    *   当前配置使用 `distribution: 'adopt'`。需要注意的是，AdoptOpenJDK 已迁移至 Eclipse Temurin。虽然 `setup-java@v4` 仍兼容 'adopt'，但长远来看，建议修改为 `distribution: 'temurin'` 以符合社区推荐标准。
*   **触发条件**：
    *   触发分支包含了 `master` 和 `main`，这是一个兼容性很好的设置。
    *   建议评估是否需要在 `pull_request` 事件中触发，因为这会消耗 CI 资源。如果是为了在 PR 合并前进行检查，这是合理的；如果只是部署，可以仅保留 `push` 触发。
*   **安全上下文**：
    *   建议检查 `CODE_TOKEN` 的权限范围，确保遵循最小权限原则，避免因 Token 泄露导致仓库被恶意修改。

### 总结
本次变更整体质量较高，是一次很好的代码清理与重构。主要的改进点在于**依赖来源的安全性校验**和**版本的确定性**。

**修改建议示例：**
```yaml
      - name: Set up JDK 11
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin' # 建议修改为 temurin
          java-version: '11'

      - name: Download Code Review SDK
        run: |
          mkdir -p ./libs
          # 建议固定版本号，避免不可控的更新
          wget -O ./libs/openai-code-review-sdk.jar https://github.com/Yaeovoi/openai-code-review/releases/download/V1.0/openai-code-review-sdk.jar
          # 如果可能，增加校验步骤
          # echo "expected-sha256  ..." | sha256sum -c
```