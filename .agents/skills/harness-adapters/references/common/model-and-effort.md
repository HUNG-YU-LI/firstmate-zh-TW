<a id="model-and-effort"></a>
# 模型與推理強度

選擇、驗證或修改任一軸前，搭配選定工具參考載入。
已設定的設定檔優先順序，另載入 `references/common/dispatch.md`。

<a id="axes-and-precedence"></a>
## 軸與優先順序

`../../../bin/fm-spawn.sh` 接受受理時選定的具體 `--harness`、`--model` 與 `--effort`；指令碼絕不解析自然語言派工規則。
工具參考記錄窗格式啟動已驗證的旗標、可接受值、省略行為與探索方式。
T3 Code 在選擇設定檔前，先依[供應商目錄契約](../../../../../docs/t3code-backend.md#provider-instances-and-models) 解析兩個軸。

推理強度優先順序為：船長對單次工作的指示、適用派工設定檔或二副固定值，最後才是下方替代規則。
絕不取代前兩種較高優先值。
只有兩者都未指定強度，才使用替代規則。

已充分理解、路徑明確且有界的工作使用 `low`；有歧義的調查或設計使用 `xhigh`。
隨複雜度、不確定性、影響範圍或開放式推理需求增加，選擇適當中間層級。
介接器若沒有 `xhigh`，使用它支援的最高非 `max` 層級，不要默默省略意圖。
替代規則絕不選擇 `max`；只有船長明確的單次或常設偏好才允許。

明確的原生 `ultra` 值，遵循 `../../../bin/fm-harness.sh validate-native-effort` 中以模型為範圍的拒絕契約；絕不默默省略，也不對應成 Pi 層級。
其他強度值的後端專屬拒絕或省略行為，包括沒有已驗證互動式強度旗標的執行環境，以 [`configuration.md`](../../../../../docs/configuration.md#crew-dispatch-profiles-configcrew-dispatchjson) 為準。
Polytoken 使用其[模型範圍內可選參考契約](../harness/polytoken.md)，不是記錄後省略。

<a id="harness-and-provider-identity"></a>
## 執行環境與供應商身分

執行環境身分獨立於模型供應商。
`harness=pi` 搭配 `model=xai/grok-*`，是 Pi 使用 xAI，不是獨立 Grok Build，也不需要登入 Grok CLI。
`harness=cursor` 搭配 `model=cursor-grok-4.5-*`，是 Cursor 路由到 Grok 模型，不是 `harness=grok`。

沒有任何指令碼會替你解析憑證來源。
應從工具探索介面與 `quota-axi auth --json` 的逐供應商來源確認，並呈現推理，不要只從名稱推測。

<a id="discovery"></a>
## 探索

將模型與供應商知識視為需要即時探索的資訊，不是永久命名空間或對應。
在目前已驗證環境中使用選定工具參考的權威介面，因為可用性會隨版本、帳號與設定改變。

遇到不熟悉的命名空間，從該執行環境的 CLI 說明、模型清單或目前文件，確認支援與供應商身分。
若已連到帳號的清單未列出某模型，就是具體不支援證據；阻擋該候選並引用證據。
無法連線的介面不能證明任何事；回報不確定性，不要下定論。

符合設定檔陣列時，只有確認每個候選的執行環境支援、供應商關係與不確定性後，才回到 `quota-array-dispatch`。
