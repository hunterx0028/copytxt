1.status為6，並發送至 as400uri：WcfServiceLib/HandleSubActService.cs

- 先做必要欄位檢核(WebNo、FunType、OpenSource、BranchId、IDNO、NTFlag/FCFlag/DVPFlag
- 三擇一 isDel/isNew/isReNew 三者擇一為Y
- 驗證失敗:回傳 ReturnCode=9999 + 對應錯誤訊息,不做任何後續動作。
- 驗證成功:回傳 ReturnCode=0000,並把整包 data 丟進 _SubServiceRecMsgQueue。尚未寫DB、也還沒送電文給 AS400。


2.CathaySec.ActIntegration.SubSendGW-master 消費此 _SubServiceRecMsgQueue


3.SendDataQueueThread.cs 為主要處理程序
3.1.依帳戶種類展開多筆：台幣、外幣、DVP
3.2.每一筆都要補資料
  - 客戶基本資料：SubCusData、SubContractSignInfo、SubCusRelatedInfoData、PL_OLAcct
  - KYC：SubCusRiskRatingData
  - 被授權人清單：SubCusAUPersonData
  - 已簽署SN商品清單：SubCusSNData
3.3.查無基本資料時退出
3.4.無紙化處理： OpenSource=5 且 isDel/isReNew=Y 時, 分別更新 PL_OLAcct/PS_OLAcct/PM_OLStatus，其中 PL_OLAcct 再寫一筆 PL_ApproveLog 審核紀錄(Action=D)
3.5.刪除處理：若 isDel=Y 且 Status=6 ，不組電文直接刪除。刪除時要把 SubCusData 搬進 SubCusData_DelInfo。(組電文刪除的狀況為 PTCTM08 銷戶電文)
3.6.新增或重傳處理：isNew=6 且 Status=6 ,或是 isReNew=Y，發送電文至 AS400
  - PTCTM03 KYC資料(若有)
  - PTCTM02 客戶基本資料
  - PTCTM18 客戶資料-原住民全銜(僅中/英文全銜其一有值才送,且會重新取號 WebSeqNo)
  - PTCTM06 被授權人資料(逐筆送)
  - PTCTM14 客戶資料-已簽署之SN商品(逐筆送)
3.7.任一電文失敗，status 更新為 E1(開戶資料送後台失敗), 成功更新為 7(開戶資料已送後台)



AS400確認資料無誤回 WOK，status 更新為 8 開戶資料寫後台成功
AS400資料審核通過回 AOK，status 更新為 9 開戶成功
AS400收到電文後後端處理回覆 ERR，status 更新為 10 後台寫檔失敗











 ▐▛███▜▌   Claude Code v2.1.228
▝▜█████▛▘  Sonnet 5 · Claude Pro
  ▘▘ ▝▝    C:\Users\leegu


❯ /cathay-integrate-account

這次想了解的是哪一類問題?

● User answered Claude's questions:
· 這次想了解國泰證複合開戶系統的哪一類問題? → 電文發送議題

好的,這次聚焦在電文發送議題,範圍限定在 sub_gateway_clean_code。請問具體想了解什麼?例如:

- 特定電文代號的內容(帶哪些欄位)
- 某支電文被閘道接收後,後續觸發了什麼動作(轉發電文/寫資料庫等)
- 特定欄位最終寫進哪個資料表

✻ Baked for 8s

❯ 在複委託開戶後，按下儲存時，會發動 HandleSubActService/InsSubActCusData ，我想知道其後的動作。

  Searched for 6 patterns, read 5 files

已找到完整的後續處理鏈路。整理如下(全部位於 sub_gateway_clean_code):

1. WCF 進入點:HandleSubActService.InsSubActCusData

WcfServiceLib/HandleSubActService.cs:13-83(對外契約定義在 IHandleSubActService.cs:13-15,WebInvoke POST /InsSubActCusData)

- 先做必要欄位檢核(WebNo、FunType、OpenSource、BranchId、IDNO、NTFlag/FCFlag/DVPFlag 三擇一有值、ModifiedUser、isDel/isNew/isReNew 且三者只能擇一為Y)。
- 驗證失敗:回傳 ReturnCode=9999 + 對應錯誤訊息,不做任何後續動作。
- 驗證成功:回傳 ReturnCode=0000,並把整包 data 丟進一個靜態記憶體佇列 _SubServiceRecMsgQueue(HandleSubActService.cs:73)。這一步只是收下電文放進佇列,尚未寫DB、也還沒送電文給 AS400。

2. 誰在消費這個佇列:SubSendGW.exe(WinForms)

這支 WCF Service 實際上是被 host 在另一個獨立程式 CathaySec.ActIntegration.SubSendGW-master(WinForms exe)裡:

- fmSubSMain.cs:233:myServiceHost = new ServiceHost(typeof(WcfServiceLib.HandleSubActService));——啟動時把上面那支 WCF Service 起起來。
- fmSubSMain.cs:59:_SendDataMsgQueue = WcfServiceLib.HandleSubueue;——把自己的佇列指標接到 Service那個靜態佇列上(這是兩個類別間的銜接點,同一個 process 內的記憶體佇列,不是跨程序通訊)。
- 接著啟動背景執行緒跑 SendDataQueueThread.DoWork(),持續 deQuecs:60-65)。
- Load 時另外會連線 AS400 DataQueue(ConnectToDataQueue(),對應 Web.config/App.config 的 ServerIP/Id/Pwd/LibraryName/SendQueueName)。

3. SendDataQueueThread.DoWork() 實際處理(SendDataQueueThread.cs)

拿到一筆 SubCusDataDO 後:

1. 依帳戶種類展開成多筆:NTFlag/FCFlag/DVPFlag 各自為 Y 就展開一筆(DVP 若 IDNOS_DVP 有逗號分隔多組檢查碼,會再展開多筆)(:38-91)。
2. 對每一筆從DB補資料(Utils.cs):
  - Utils.GetSubBaseData — 查 SubCusData 表(join SubContractSignInfo、SubCusRelatedInfoData、PL_OLAcct)取客戶基本資料
  - Utils.getSubKYCData — 查 SubCusRiskRatingData 取KYC
  - Utils.getSubAUPersonData — 查 SubCusAUPersonData 取被授權人清單
  - Utils.GetSubCusSNData — 查 SubCusSNData 取已簽署SN商品清單
3. 查無基本資料就記 log、通知UI、continue(不往下走)(:111-116)。
4. 無紙化特殊處理(OpenSource=5 且 isDel/isReNew=Y 時,:156-246)/PS_OLAcct/PM_OLStatus 的狀態欄位,PL_OLAcct 情境下還會多寫一筆PL_ApproveLog 審核紀錄(Action=D)。
5. 刪除/重傳分支(:287-339):
  - 若 isDel=Y 且原 Status 為 6 或 E1:不組電文,直接呼叫 Utils.DeleteTables 刪除
SubCusRiskRatingData/SubContractSignInfo/SubCusRelatedInfoDataata,並把 SubCusData 搬進 SubCusData_DelInfo後刪除原資料列,continue。
  - 其餘情況:組【刪除客戶基本資料】電文 PTCTM07(DataQueueProceAS400 DataQueue。
6. 新增/重傳分支(isNew=Y 且 Status=6,或 isReNew=Y,:341-593):依序組送多支電文到 AS400 DataQueue(fmSubSMain.SendDQ.Write):
  - PTCTM03 KYC資料(若有)
  - PTCTM02 客戶基本資料
  - PTCTM18 客戶資料-原住民全銜(僅中/英文全銜其一有值才送,且會
  - PTCTM06 被授權人資料(逐筆送)
  - PTCTM14 客戶資料-已簽署之SN商品(逐筆送)

任一支電文送失敗,會把該筆 SubCusData.Status(或 KYCStatus)更新 斷該筆處理,但不影響其他筆。
7. 全部電文送成功後,呼叫 Utils.UpdSubCusData(subCusData, "ALL") 把 SubCusData.Status/KYCStatus 更新為 7,表示這筆已送出完成。
8. 若寫 AS400 DataQueue 當下發現未連線,DataQueueWrite(:613-704重試,不會遺失。

小結(銜接點與範圍界線)

- 電文最終目的地是 AS400 的 DataQueue(SendDQ.Write),不是 SQL S在這裡的角色是「電文內容的資料來源」以及「處理狀態(Status/KYCStatus)、審核記錄(PL_ApproveLog)、進件狀態(PL_OLAcct/PS_OLAcct/PM_OLStatus)的紀錄表」。
- 真正「AS400 收到電文後怎麼入帳」屬於 AS400 主機端邏輯,不在這。
- 我還沒有讀取 DataQueueProcess.cs(組電文 Structure 的細節,例如 PTCTM02/03/06/14/18 各自帶哪些欄位、Byte
Layout)。如果你想知道電文本身的欄位內容,我可以接著查這個檔案。

✻ Worked for 1m 35s

※ recap: You're tracing what happens after 複委託開戶 saves and SubActCusData in sub_gateway_clean_code; I've mapped the fullqueue-to-AS400 flow. Next: read DataQueueProcess.cs if you want the actual field-level content of each PTCTM electronic message. (disable recaps in
  /config)

❯  PL_OLAcct/PS_OLAcct/PM_OLStatus 各是什麼內容？

  Searched for 3 patterns, read 1 file

我從三個資料表相關的程式碼(DataContract 定義 + 實際SQL)分別整理如下。先說明:這三個字首(PL/PS/PM)在程式碼裡都沒有註解說明業務意涵,我是從分流條件反推使用情境,實際代表什麼我不確定,建議你確認(依 skill
規則,代號意涵不用猜的)。

PL_OLAcct

DO 類別定義:WcfServiceLib/IHandleSubActService.cs:845-948(clas

┌───────────────────────────────────────────────────────┬─────
│                         欄位                          │   說明(來自程式註解)   │
├───────────────────────────────────────────────────────┼─────
│ Pk_No                                                 │ 案件序號               │
├───────────────────────────────────────────────────────┼─────
│ BranchId                                              │ 開戶分公司(訂約分公司) │
├───────────────────────────────────────────────────────┼─────
│ IDNO                                                  │ 身份證號/統一編號      │
├───────────────────────────────────────────────────────┼─────
│ Account                                               │ 投資人戶號             │
├───────────────────────────────────────────────────────┼─────
│ NTStatus                                              │ 台幣開戶作業(狀態)     │
├───────────────────────────────────────────────────────┼─────
│ FCStatus                                              │ 外幣開戶作業(狀態)     │
├───────────────────────────────────────────────────────┼─────
│ NT_CG_ActionMode / FC_CG_ActionMode                   │ (無註解)               │
├───────────────────────────────────────────────────────┼─────
│ NT_CG_ReplaceDate / FC_CG_ReplaceDate                 │ (無註解)               │
├───────────────────────────────────────────────────────┼─────
│ NT_CG_RepNo / FC_CG_RepNo                             │ (無註解)               │
├───────────────────────────────────────────────────────┼─────
│ MSStatus                                              │ 進件狀態               │
├───────────────────────────────────────────────────────┼─────
│ CreateUser / CreateDate / ModifiedUser / ModifiedDate │ 建檔/異動人員與時間    │
└───────────────────────────────────────────────────────┴─────

注意:DO 類別不完整——Utils.UpdPL_OLAcct(Utils.cs:1123-1171)的 U 宣告的欄位OACFlag、NTBankActNo、FCBankActNo,代表實際資料表欄位比這個 DO 類別多,DO 只是 GW 需要用到的子集。

使用情境(SendDataQueueThread.cs:161-209):當開戶來源不是「未成年開戶」且 OpenSource != "A" 時走這條路(涵蓋
5.無紙化進件、6.線上開戶、7/8.載具開戶等)。Utils.GetSubBaseDat 反查 PL_OLAcct.Pk_No/OCSNo/OpenSource(Utils.cs:580)。

PS_OLAcct

DO 類別定義:WcfServiceLib/IHandleSubActService.cs:979-1180(cla比 PL_OLAcct 多,幾乎是完整申請單資料而不只是狀態:

- 狀態欄位:Pk_No、BranchId、IDNO、NTStatus、FCStatus、MSStatus
- 客戶個資:CustName、CusEName、Sex、Nationality、BDate、戶籍地址(ZipCode/AD)、通訊地址(CZipCode/CAD)、Tel/Mobile/Email/Fax
- 公司/職業:CompName、CompTel
- 銀行:NTBankNo/NTBankActNo、FCBankNo/FCBankActNo
- 業務相關:OSales、CSNo/CSBranchNo、SendAddr、SendType、緊急聯Related)
- 徵信/風險:Income、PropertyValue、Experience、RiskLevel 類欄位(WhyOpenAcc、MSource等)
- 洗錢防制:AMLJobB/AMLJobS/AMLPosition
- 其他旗標:OACFlag、Invalid_Flag、ContractSignFlag、FCAuth/NTAuth/AccAuth 等

使用情境(SendDataQueueThread.cs:210-227):當 objSubCusData.OpenSource == "A"(程式內註解確認 = 「開戶網」)時走這條路,只更新
NTStatus/FCStatus/MSStatus(Utils.UpdPS_OLAcct,Utils.cs:1227-12

PM_OLStatus

這個程式包裡完全沒有對應的 DataContract/DO 類別,只在 SQL 字串 :

- PM_OLStatus表:SeqNo(PK)、IDNO、NTStatus、MSStatus、ModifiedUser、ModifiedDate(Utils.cs:1444-1457;SubReceiveGW/Utils.cs:2787-2796還多顯示一個更新來源,同樣欄位)
- 關聯表 PM_OLAccount:Pk_No、IDNO、BranchId、CG_RepNo(SubSendGveGW/Utils.cs:2047)。兩表透過 PM_OLStatus.SeqNo =PM_OLAccount.Pk_No 關聯。

使用情境(SendDataQueueThread.cs:228-243):上面兩個條件都不成立時(else 分支)才會呼叫 Utils.UpdPM_OLStatus。往前追這個 else 分支的觸發條件是 OpensourceDesc
== "未成年開戶"(即 OpenSource == "9" 且 AUPersionList.Count >  ),所以 PM很可能是「未成年(Minor)」相關的意思——但這是我從分流邏輯推測,程式裡沒有直接註解證實,想跟你確認一下這個猜測對不對。

---
想跟你確認幾點,好讓後續分析/重構更準確:
1. PL / PS / PM 這三個字首各自代表什麼業務名稱(例如是否為系統模組縮寫)?
2. 是否有其他文件或 DB schema 可以核對 PL_OLAcct 完整欄位(因為只是子集)?

✻ Crunched for 1m 27s

※ recap: We're tracing what happens after 複委託開戶 save triggeg the AS400 electronic-message flow and status-table updates(PL_OLAcct/PS_OLAcct/PM_OLStatus). Next step is for you to confirm what the PL/PS/PM prefixes actually mean. (disable recaps in /config)

❯ 目前我已知status為6時是開戶資料儲存成誤但未送出，7是開戶資料已送台，E1是開戶資料送後台失敗。那其他status的10,8,9,D各會在何時寫入？

  Searched for 5 patterns, read 2 files

已在 sub_gateway_clean_code 的收電文流程(SubReceiveGW/SubReceiveQGW 兩個專案)裡找到這些 Status 值的寫入時機。整理成完整表格(含你已知的
6/7/E1),並標明每個值的觸發點:

完整 Status 對照表(SubCusData.Status)

Status: 6
觸發時機: (已知)開戶資料儲存但未送出
寫入位置: integrate_account 端存檔時寫入(不在此程式包範圍)
────────────────────────────────────────
Status: 7
觸發時機: (已知)開戶資料已送台(GW組電文+送AS400 DataQueue 全部成功)
寫入位置: SubSendGW/SendDataQueueThread.cs:591 Utils.UpdSubCus
────────────────────────────────────────
Status: E1
觸發時機: (已知)開戶資料送後台失敗——組電文失敗或送 DataQueue 失敗(GW端這一關就沒送出去)
寫入位置: SubSendGW/SendDataQueueThread.cs,例如 :407(組客戶基 料電文失敗)等多處
────────────────────────────────────────
Status: 8
觸發時機: 電文已送達 AS400,且 AS400 回覆 ReturnCode = OK 或 WOK(電文格式/寫檔正確,但客戶主檔還在等後台審核)
寫入位置: SubReceiveQGW/fmSubQMain.cs:292(PTCTM03)、:333(PTCTMPTCTM14)、:463/466(PTCTM18)
────────────────────────────────────────
Status: 9
觸發時機: 兩種情境:① PTCTM02(客戶主檔)收到 AS400 回覆 ReturnCode = AOK(後台審核通過,開戶正式生效),同時 KYCStatus 也設為 9;② PTCTM08 銷戶事件電文中
EventKind=D 且 FunType=D(取消銷戶/復戶)時,把原本 D 改回 9,CusS
寫入位置: ①SubReceiveQGW/fmSubQMain.cs:335-338;②SubReceiveGW/Utils.cs:1074-1077
────────────────────────────────────────
Status: 10
觸發時機: 電文送達 AS400 後,回覆 ReturnCode = ERR(AS400端業務  ——E1是GW這邊沒送出去,10是送到了但AS400退件)
寫入位置: SubReceiveQGW/fmSubQMain.cs:295(PTCTM03)、:341(PTCTM02)、:365(PTCTM06)、:443(PTCTM14)、:469(PTCTM18)
────────────────────────────────────────
Status: D
觸發時機: PTCTM08 銷戶事件電文,EventKind=D 且 FunType=A(執行銷成 D、CusStatus='02'、SubCanCelDate=銷戶日
寫入位置: SubReceiveGW/Utils.cs:1014-1018

補充說明

- 8→9 是一組流程:電文送到 AS400 先回 WOK(格式/寫檔正確,狀態先標 8),AS400 內部後台審核通過後才會再送一次 AOK 回覆,GW 收到才把狀態改成 9。所以 8
是「已送達待審」,9 才是「審核通過/正式完成」。
- E1 vs 10 的差異:E1 是 SubSendGW(送電文端)自己組電文失敗或寫 AS400 DataQueue 失敗,電文根本沒送出去;10 是電文確實送到 AS400,但 AS400 端處理後回覆 ERR,由
SubReceiveQGW(收電文端)寫回的狀態。兩者是完全不同的專案/流程在
- D 是銷戶專用狀態,由 PTCTM08 這支「事件電文」驅動(不是 PTCTM02 客戶主檔電文),且會連動更新 PL_OLAcct/PM_OLStatus/PB_OLStatus/PS_OLAcct 的 MSStatus 欄位為
D(Utils.cs:1019-1059),這也印證了你上一題問的那三張表是同一批「penSource)分流寫入不同表。
- PTCTM07(刪除電文)本身沒有設定 Status——fmSubQMain.cs:370-391 只解析回覆內容,實際刪除/搬移動作是在 ReturnCode=WOK 且 FunType=D
時另外處理(SubReceiveGW/Utils.cs:237-…,搬移到 SubCusData_DelIn 没有 Status 可言)。

想確認一下:這個 Status 欄位(以及對照表)是否就是你要重構評估的 查 KYCStatus(目前只確認會跟 Status 同步設8/9/10,但也有獨立設定的地方,如 PTCTM03 只更新 KYCStatus 不動 Status)的完整規則,一起列出來。

✻ Cooked for 1m 44s

※ recap: We're mapping the sub_gateway_clean_code 電文發送流程 for the IASCUS013 複委託開戶 refactor, and just decoded SubCusData.Status codes 8/9/10/D.
  Next: confirm whether you want KYCStatus's full rules documentconfig)

❯ 我想了解 status=6 當下寫入SubCusData, 以及AS400回 WOK 或 AOK   值調整

  Read 1 file

我先回答 AS400 回 WOK/AOK 時 SubCusData 的寫入細節(這仍在 sub_ Status=6這筆資料的「當下寫入」動作,是在使用者於畫面按下儲存的當下就發生了,那是 integrate_account_clean_code
的職責範圍,不在我們目前設定的「電文發送議題」範圍內——我在 sub_QueueThread 讀取已存在的 Status='6'資料列(當作送電文的前置條件),沒看到 6 是在哪裡被寫入的。

AS400 回 WOK / AOK 時,SubCusData 的寫入方式(PTCTM02,客戶主檔電文)

實際執行的是同一段 SQL(SubReceiveGW/Utils.cs:187),WOK 和 AOK 只差在傳入的 @Status 參數值不同:

Update SubCusData
Set ElecConsent   = case when ETradingCode = '0' then getdate(
    InternetBank   = case when ETradingCode = '0' then 'Y' else InternetBank end,
    Account        = @Account,
    ApproveDate    = convert(datetime, @ApproveDate),
    ApproveBranchId= @ApproveBranchId,
    ApproveBoss    = @ApproveBoss,
    Status         = @Status,
    KYCStatus      = @Status,      -- 注意:跟 Status 綁同一個參數
    ModifiedUser   = @ModifiedUser,
    ModifiedDate   = GetDate(),
    SubOpenDate    = GETDATE()
Where IDNO = @IDNO and BranchId = @BranchId;

鍵值(WHERE 條件):IDNO + BranchId(複合鍵,與先前看到的其他 Status 更新語句一致)。

參數怎麼來的(fmSubQMain.cs:300-343 解析 AS400 回覆電文 → Utils.UpdActInfoUploadStatus/UpdActInfo,參數組裝於 Utils.cs:1125-1140):

┌────────────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────────────┐
│                    欄位                    │                出的值)                               │
├────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤
│ @Account                                   │ AS400回覆的 H24 的戶號」,審核後由AS400配發回來才寫入 │
├────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤               │ @ApproveDate/@ApproveBranchId/@ApproveBoss │ AS400回覆的 H252(核准主管)                           │
├────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤               │ @Status                                    │ WOK → "8";AOK →                                      │
├────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────┤               │ @ModifiedUser                              │ 固定字串 "SubRe                                      │
└────────────────────────────────────────────┴───────────────────────────────────────────────────────────────────────────────────┘               
WOK vs AOK 差在什麼:                                                                                                                             - WOK(fmSubQMain.cs:332-334):代表 AS400 已收到電文、格式檢核/ 審核。此時 Status 和 KYCStatus 都被寫成 8,同時Account(戶號)欄位也會被 AS400 回傳值覆寫——也就是說戶號在 WOK 階段就已經配發並寫回,不是等到 AOK 才有。                                            - AOK(fmSubQMain.cs:335-338):代表後台審核通過,Status/KYCStatusproveBranchId/ApproveBoss(審核日期/分公司/主管)、SubOpenDate(開戶日,GETDATE(),即GW收到AOK當下的系統時間)也一併寫入。另外還有個附帶邏輯:ElecConsent(電子同意書/開戶日)與 InternetBank(是否申請網銀)欄位,只在 ETra= '0'(可買賣)時才會被設成 GETDATE()/'Y',否則維持原值不動。
                                                                                                                                                 ---
要不要我把範圍切到 integrate_account_clean_code,幫你追「使用者按儲存時,SubCusData 是怎麼被 INSERT 成 Status=6」的那段程式(欄位、Key、是哪個 Svc/D
✻ Crunched for 57s                                                                                                                                 
❯ 不用切換 integrate_account_clean_code，我想了解的是 WOK 等於AS400傳來了account，此時SubCusData是刪除一筆record，但依據accountl新增？還是直接updat
不是刪除再新增,是直接 UPDATE 同一筆既有的 record。                                                                                               
確認:單一 UPDATE 語句,Key 不是 Account                                                                                                           
SubReceiveGW/Utils.cs:187(PTCTM02 case,WOK 和 AOK 共用同一段 SQL):                                                                               
Update SubCusData                                                                                                                                Set ElecConsent   = case when ETradingCode = '0' then getdate(
    InternetBank   = case when ETradingCode = '0' then 'Y' else InternetBank end,                                                                    Account        = @Account,      -- ← Account 只是「被寫入
    ApproveDate    = convert(datetime, @ApproveDate),                                                                                                ApproveBranchId= @ApproveBranchId,
    ApproveBoss    = @ApproveBoss,                                                                                                                   Status         = @Status,
    KYCStatus      = @Status,                                                                                                                        ModifiedUser   = @ModifiedUser,
    ModifiedDate   = GetDate(),                                                                                                                      SubOpenDate    = GETDATE()
Where IDNO = @IDNO and BranchId = @BranchId;   -- ← 真正的查找/鎖定鍵值                                                                          
關鍵點:WHERE 子句用的是 IDNO + BranchId,不是 Account。這代表:                                                                                    
1. 這一筆 record 早在使用者存檔(Status=6)時就已經用 IDNO + BranchId 當識別鍵存在於 SubCusData 了。                                               2. WOK 回覆帶來的 Account(戶號)是 AS400 那邊新配發的值,GW 只是t 欄位,不會、也不需要拿 Account 去找row——因為在這個時間點之前,這筆 record 本來就沒有 Account 值(開戶送出時通常是空的,審核/配號後才第一次填入)。                                      3. 整個過程完全沒有 DELETE 或 INSERT INTO SubCusData,IDNO+Bran7)→AS400回覆(8/9)全程沒有變動,只是同一筆 row的欄位被陸續補齊/更新。                                                                                                                          
唯一有 INSERT 動作的地方,是另一張表(不是 SubCusData)                                                                                             
同一個 case 區塊裡還有一段 INSERT INTO dbo.AcctInfoByID(Utils.cs:200-224)——這是「歸戶檔」,是一張獨立的彙總表,記錄「這個 IDNO 底下有哪些帳號」,跟 SubCusData 本身(客戶主檔)是兩張不同的表:

INSERT INTO dbo.AcctInfoByID
   (IDNO, BranchId, Account, Type, ODate, CDate, Status, CreateUser, CreateDate, ModifiedUser, ModifiedDate)
VALUES
   (@IDNO, @BranchId, @Account,
    (Select Top 1 Acttype from SubCusData with(nolock) where IranchId),
    @ODate, NULL, 'Y', 'GW-Insert', @ApproveDate, 'GW-Insert', @ApproveDate)

這筆 INSERT 是新增一筆歸戶紀錄(用來讓其他系統查「某身分證底下所有帳號」),跟 SubCusData 本身的 UPDATE 是各自獨立的動作,同一個 SQL 批次(sql.AppendLine
累加成一個 StringBuilder,Utils.cs:1141 一次 ExecuteNonQuery 整 刪 SubCusData 再依 Account 新增」這種模式。

✻ Brewed for 28s

❯ 但我看SubCusData的鍵值是 BranchId, Account, IDNO。雖然status=6AS400也不能直接更新這個鍵值啊

  Searched for 2 patterns

你提的疑慮是合理的,但要拆成兩件事看:

1. 這個 UPDATE 本身,查找用的 key 是什麼

我從程式碼(SubReceiveGW/Utils.cs:187 以及所有前後手動作,如 Sen

- Status 還在 6/7/8(尚未核准配號)階段,所有對 SubCusData 的查找 + BranchId,從沒出現過 Account 當查找條件(因為這階段 Account根本還是空的,拿它當查找條件也查不到)。
- 只有到 PTCTM08 事件電文(銷戶/過戶,Utils.cs:1013/1018/1049/10 發生的動作,才會改用 BranchId + Account 當查找條件。

所以就 WOK 這支 UPDATE 而言,它是拿 IDNO + BranchId 去鎖定 row  進去——它沒有拿 Account當查找依據,自然也不會發生「用還沒配到的空白 Account 去找 row 結果找不到」的問題。

2. 如果 SubCusData 的實體 PK 真的是 (BranchId, Account, IDNO) 這組複合鍵

這是你提出的關鍵疑點——如果 Account 確實是複合主鍵的一部分,SQL Server 從語法上仍然允許直接 UPDATE 這個欄位值(不像 IDENTITY 自動編號欄位那樣被鎖死),只要:
- 更新後不會跟其他既存 row 的鍵值組合重複(唯一性沒被破壞——這裡 別人)。
- 沒有其他資料表用 FK 外鍵約束卡住這張表、且未設定 ON UPDATE CASCADE。

但我必須老實說:我在這兩包程式碼裡完全找不到 SubCusData 的 CREATE TABLE/主鍵/約束定義——如同這系統的架構特性(這套系統沒有獨立 .sql 或 DB schema
檔案),我只能看到 DAO 裡組的 SQL 語句,看不到實際建表時宣告的 PK 以:

- 我無法從程式碼證實或推翻「SubCusData 的 PK 是 (BranchId, Acc
- 這段 UPDATE 語句能不能成功執行(不噴 PK 違反錯誤),取決於資料庫實際的約束定義,而不是我目前查得到的東西。

想請教一下:你看到「鍵值是 BranchId, Account, IDNO」這個資訊的來源是什麼(例如 DBA 提供的 schema 文件、或是 SSMS 裡直接看到的索引定義)?如果你能提供實際的
PK/Unique Index 定義,我可以幫你比對程式碼裡的這些 UPDATE/SELEC 風險——這對你要重構的部分應該會是一個重要的風險點。

✻ Brewed for 1m 3s

※ recap: We're analyzing Cathay's 複委託開戶 message-dispatch flor of IASCUS013. Right now we're pinned down on whetherSubCusData's real primary key includes Account, since no DDL exists in the code, waiting on you to share the actual schema/index definition.             
❯ 什麼時候會更新 SubCusData 的 account ？                                                                                                                  
  Searched for 4 patterns, read 1 file                                                                                                                     
我把整個 sub_gateway_clean_code 搜過一輪(Account = @Account,、Account = @New_Account 全資料夾比對),SubCusData.Account 欄位只在兩種情境下會被寫入:        
情境 1:PTCTM02 電文回覆 WOK 或 AOK(你目前在討論的這個)                                                                                                   
SubReceiveGW/Utils.cs:187——UPDATE SubCusData SET Account = @Account ... WHERE IDNO=@IDNO AND BranchId=@BranchId。這是 Account                            第一次從空白被寫入正式戶號的地方,前面已經討論過。
                                                                                                                                                         同一批 SQL 裡,還會同步把 Account 寫進其他關聯表(用同一個 @Acco 查找鍵):
- SubCusRiskRatingData(:189)、SubCusRelatedInfoData(:191)、SubCusAUPersonData(:193)、SubContractSignInfo(:195)、SubCusSNData(:197)                       
情境 2:PTCTM08 事件電文——過戶轉分公司/換戶號(EventKind='F' 且 FunType='A')                                                                               
這個情境不是去 UPDATE 同一筆 row 的 Account,而是「開一筆新 row + 把舊 row 標記關閉」兩步驟:                                                              
Step 1 — 複製一筆新 row(新分公司+新戶號):Utils.cs:455-…                                                                                                  INSERT INTO dbo.SubCusData (BranchId, Account, IDNO, ... 其餘
Select BranchId = @New_BranchId                                                                                                                                ,Account  = @New_Account
      ,IDNO     = @IDNO                                                                                                                                        ,... (其餘欄位大多原封複製,但 StkCusKind/GroupNo/OSales/計算)
From SubCusData Where BranchId = @Old_BranchId and Account = @Old_Account;                                                                               
Step 2 — 把舊 row 標記關閉(不是刪除):Utils.cs:1014-1018                                                                                                  Update SubCusData
Set IDNO = @New_IDNO, CusStatus = '02', Status = 'D', SubCanCelDate = Convert(Date, @CDate),                                                                 ModifiedUser = @ModifiedUser, ModifiedDate = GetDate()
Where BranchId = @Old_BranchId and Account = @Old_Account;                                                                                               
換句話說:「過戶」在這裡的實作是「新開一筆帳號 row + 把舊帳號 row 的 Status 改成 D(銷戶/關閉狀態)」,舊 row 的 Account 欄位值本身從頭到尾沒被改過,只有它的 IDNO(改成 @New_IDNO)和 Status 被動了。這跟情境 1(同一筆 row 內的兩種操作模式。

小結對照你上一題的疑慮

這也剛好回答了你上一題「Account 是否是複合鍵」的疑慮:程式邏輯   Account到另一個值」這種操作——需要換戶號/換分公司時,寫的是「新增新 row + 舊 row 標記 D」,而不是「UPDATE 舊 row 的 Account 欄位」。唯一一次 UPDATE
Account,是從空白填成第一次配發的正式值(情境1),而不是從一個已存 身就很像是在刻意迴避「更新複合鍵欄位」的風險,間接支持了Account 可能真的是鍵值一部分的猜測——但如前所述,我仍然無法從程式碼裡的 DDL 證實這點。

✻ Baked for 1m 58s

❯ WOK時，會更新Account，AOK時，會更新ApproveDate/ApproveBranchId/ApproveBossaccount，是這樣的嗎？

不完全正確。從程式碼看,這不是「WOK 更新 Account、AOK 更新 Approve 相關欄位」這種條件式分工,而是:

實際邏輯:同一段 SQL,不論 WOK/AOK(甚至 ERR)都會執行

SubReceiveGW/Utils.cs 裡 case "PTCTM02": 這段(:181-229)沒有針對 ReturnCode 做任何 if 判斷,不像 PTCTM07 那樣有 if (obj.ReturnCode == "WOK" && obj.FunType
== "D") 的條件包裹。也就是說,只要收到一支 PTCTM02 回覆電文,不  ERR,都會執行同一段 UPDATE,同時寫入Account、ApproveDate、ApproveBranchId、ApproveBoss、Status、KYCStatus。

而且這些值(_AS400.Account/_AS400.ApproveBranchId/_AS400.ApproveBoss)在 fmSubQMain.cs:303-321 的欄位解析階段,是在 switch(_AS400.ReturnCode) 判斷 Status
之前就已經無條件解出來了(H24→Account、H31→ApproveBranchId、H32  WOK 還是 AOK 完全無關。

真正因 WOK/AOK 而不同的,只有這件事

fmSubQMain.cs:330-341:

switch (_AS400.ReturnCode)
{
    case "WOK":
        _AS400.Status = "8";
        break;
    case "AOK":
        _AS400.KYCStatus = "9";   // 這行其實沒被用到,見下方說
        _AS400.Status = "9";
        break;
    case "ERR":
        _AS400.Status = "10";
        break;
}

唯一真正依 WOK/AOK/ERR 而變的欄位值只有 Status(連帶因為 SQL 裡參數,KYCStatus 也跟著變成同一個值——_AS400.KYCStatus = "9"這行程式碼本身在 DB 寫入時沒被實際引用,因為 Utils.cs:1125-1138 組參數時只用了 obj.Status,沒有 obj.KYCStatus 這個參數,這行等於是寫假的死碼)。

一個值得注意的疑點:ApproveDate 的算法

ApproveDate 綁定的值不是直接從電文欄位取得,而是 Utils.cs:1109-1123 這段邏輯算出來的:

DateTime approveDate = DateTime.Now;
if (obj.ReturnDate != null && obj.ReturnTime != null)
{
    if (obj.ReturnDate.Length == 8 && obj.ReturnTime.Length ==
        && obj.ReturnDate != "00000000" && obj.ReturnTime == "000000")
    {
        approveDate = new DateTime(...這裡才用 ReturnDate/ReturnTime 組出日期...);
    }
}

這裡的條件是 obj.ReturnTime == "000000" 才會採用電文帶回來的日期時間,否則一律預設成 DateTime.Now(GW
收到回覆當下的系統時間)。這個條件寫法看起來怪(一般會預期是「有間剛好等於000000」才採用),我不確定是不是刻意設計還是筆誤,如果你要重構這段,建議特別留意這個判斷式,可能要跟原開發人員或需求文件核對意圖。

小結回答你的問題

你原本的理解(WOK 專門更新 Account、AOK 專門更新 Approve 相關欄位)在 GW 端程式碼裡並不存在這種切分——這幾個欄位是每次收到 PTCTM02
回覆就整批一起更新,差別只在於：① Status/KYCStatus 的值依 WOK(8ount/ApproveBranchId/ApproveBoss這些欄位實際帶什麼值(是有意義的資料,還是空白/預設值),完全取決於 AS400 host 端在 WOK 階段和 AOK 階段各自往這些欄位放了什麼——這部分屬於 AS400
主機邏輯,不在這兩包程式碼的可查證範圍內,如果你要確認「WOK 階段號、AOK 階段才給核准資訊」,會需要 AS400那邊的電文規格文件或跟後台人員確認。

✻ Crunched for 1m 21s

※ recap: We've been tracing 複委託開戶 status flow for a reconfirmed reference. I just corrected your assumption that WOK/AOK update different SubCusData
  fields separately — they run the same UPDATE together. Let me

❯ 我確認了 SubCusData 的 鍵值是 BranchId, Account,IDNO。既然AS400會更新Account，那為什麼還要把Account設為鍵值？還是程式中有什麼地方，是利用這三個組合鍵值查資料或更新資料的？

  Searched for 1 pattern, read 1 file

找到了。程式碼裡確實有地方是拿 BranchId + Account + IDNO 三欄位組合當作查找/更新鍵值,而且是同一張 SubCusData 表:

確實存在的三鍵組合用法

CathaySec.ActIntegration.SubPTCWA00SendGW-master/CathaySec.ActIntegration.SubPTCWA00SendGW/Utils.cs:290-333,函式
UpdateSubCusDataFromPTCTM15(註解:「跨點代做-客戶個資變更」),觸

update SubCusData
set Tel=@Tel, Addr=@Addr, CAddr=@CAddr, CZipCode=@CZipCode, Mobile1=@Mobile1,
    Email=@Email, Fax=@Fax, CompTel=@CompTel, SealCDate=@SealC
    EAddr=@EAddr, ECAddr=@ECAddr, ZipCode=@ZipCode, CensusTel=@CensusTel, Mobile2=@Mobile2
where BranchId = @BranchId and Account = @Account and IDNO = @

參數 @BranchId/@Account/@IDNO 都是直接從 PTCTM15 這支電文帶進 count/data.IDNO,:306-308),用來更新戶籍/通訊地址、電話、Email等聯絡資料欄位。

為什麼這裡可以、也需要用三欄位當鍵值

我從程式碼的使用時機差異觀察到一個規律,可以合理解釋你的疑問:

┌──────────────────────────────────────────────────┬────────────────────────────────────────────────────────────┬─────────────────────────────────────┐
│                       階段                       │          a                    │         Account 當下的狀態          │
├──────────────────────────────────────────────────┼────────────────────────────────────────────────────────────┼─────────────────────────────────────┤
│ 開戶申請中(Status 6/7/8,尚未 AOK)                │ 只用 IDNO  SendGW/ReceiveGW   │ 還是空白,不能拿來當查找條件         │
│                                                  │ 邏輯)                                                      │                                     │
├──────────────────────────────────────────────────┼───────────────────────────────┼─────────────────────────────────────┤
│ 開戶已核准後的維護作業(如 PTCTM15                │ BranchId + Account + IDNO 三欄位一起                       │ 已經是 AS400                        │
│ 跨點代做個資變更)                                │                               │ 配發的正式戶號,是穩定值             │
└──────────────────────────────────────────────────┴────────────────────────────────────────────────────────────┴─────────────────────────────────────┘

也就是說:這套系統本身就分成「申請中」跟「已核准」兩個階段在用不同的鍵值組合查資料——申請中階段 Account 是空的沒得用,只能靠
IDNO+BranchId;帳號核准、正式戶號寫入之後,後續所有「針對這個已 M15 這種個資變更),就會連 Account一起帶進來精準卡到那一個特定帳戶。

這樣回頭看,為什麼 Account 要放進實體 PK 就說得通了

如果實體資料表的 PK 真的是 (BranchId, Account, IDNO),這個設計其實有個合理用意:

- 申請中的那一筆 row,Account 是空字串 ''。PK (BranchId, '', IDNO) 只會擋住「同一個 IDNO+BranchId 同時有兩筆都還在申請中(Account 皆為空)的
row」——這剛好符合業務規則「同一人同一分公司,同時間只能有一件複
- 一旦核准,Account 從 '' 被 UPDATE 成真正戶號,這筆 row 的 PK 值就從 (BranchId, '', IDNO) 變成 (BranchId, '正式戶號', IDNO)——如同上一輪討論的,SQL Server
允許直接 UPDATE 複合 PK 裡的欄位值,只要新值不撞到既有 row。
- 因為 Account 是 PK 的一部分,同一個 IDNO+BranchId 底下,理論上可以並存多筆「不同 Account」的歷史 row(例如銷戶後又重新開戶、或前面看到的過戶事件 INSERT
INTO SubCusData ... Account=@New_Account 產生的新 row),彼此不  唯一性——因為它們的 Account 值不同。

這是我從程式碼使用模式反推出來的合理解釋,不是我從 schema 文件 跟提供 PK 資訊給你的來源(DBA/schema文件)核對一下,尤其要確認：「申請中階段,Account 空字串是否真的算進 PK 唯一性檢查」這件事,是資料庫層級真的有這樣的約束,還是只是程式邏輯自己假設「同一人同分
公司只會有一筆進行中案件」、但資料庫其實沒有真的擋——如果是後者別注意的資料完整性風險點(因為 Status=6 那筆的 INSERT 邏輯在integrate_account_clean_code,我們目前沒有查證那邊有沒有做這個唯一性檢查)。

✻ Sautéed for 1m 39s

❯ 查找一下並彙整，何時會有相同 BranchId + IDNO, 但不同 Account 的狀況出現？

  Searched for 1 pattern, read 1 file

我查了 SubReceiveGW/Utils.cs 裡所有跟 SubCusData.IDNO 改寫相關的邏輯,並比對 SubSendGW 端怎麼查資料,整理出結論:目前程式碼裡只有一種情境會產生「相同
BranchId+IDNO、不同 Account」的多筆 row,而且這個情境本身還牽動提醒你。

情境 A(唯一確認會發生的路徑):銷戶後,同分公司重新開戶

PTCTM08 事件電文,EventKind='D'(銷戶)且 FunType='A'(執行銷戶), Utils.cs:1014-1018):

Update SubCusData
Set CusStatus = '02', Status = 'D', SubCanCelDate = Convert(Date, @CDate), ModifiedUser = @ModifiedUser, ModifiedDate = GetDate()
Where BranchId = @BranchId and Account = @Account;

注意:這裡完全沒有動到 IDNO 欄位。 舊帳戶 row 只是把 Status 改  者原封不動留在表裡。

如果客戶之後在同一分公司重新申請複委託開戶(在 integrate_accoununt 又是空白開始),沿用同一個IDNO+BranchId,新案件跑完整流程核准後,新 row 的 Account 被 AS400 配上一個新戶號。此時表裡就會同時存在:

┌──────────────┬──────────────┬─────────────┬─────────┬─────────┐
│     Row      │   BranchId   │    IDNO     │ Account │ Status
├──────────────┼──────────────┼─────────────┼─────────┼─────────┤
│ 舊(已銷戶)   │ 同一家分公司 │ 同一組 IDNO │ 舊戶號  │ D
├──────────────┼──────────────┼─────────────┼─────────┼─────────┤
│ 新(重新開戶) │ 同一家分公司 │ 同一組 IDNO │ 新戶號  │ 6→7→8→
└──────────────┴──────────────┴─────────────┴─────────┴─────────┘

系統設計者顯然也意識到這個問題——但只在「有條件」下處理

在同一個 EventKind='D'/FunType='A' 區塊裡,緊接著有這段(Utils.cs:1061-1064):

if (!string.IsNullOrEmpty(obj.New_IDNO))
{
    UpdateNewIDNO(sql);
}

UpdateNewIDNO(Utils.cs:1175-1185)會把這筆剛關閉的 row 的 IDNO yFlag = 'E':

Update SubCusData Set IDNO = @New_IDNO, ModifiedUser = @Modifie(), isCopyFlag = 'E' Where IDNO = @IDNO;

這其實就是「主動騰空原 IDNO+BranchId 組合」的補救機制——但它只   New_IDNO 值時才會執行。也就是說:
- AS400 有給 New_IDNO → 舊 row 的 IDNO 被改走,原 IDNO+BranchId 組合被騰空,之後重開戶不會撞鍵 → 不會出現同 BranchId+IDNO 不同 Account。
- AS400 沒給 New_IDNO(推測是較常見的一般銷戶) → 舊 row 原封不 ranchId+IDNO 不同 Account。

反例:過戶轉分公司事件(EventKind='F')——不會產生這個情況

Utils.cs:986 過戶時,舊分公司的 row 一律會把 IDNO 改寫成 @New_I 改):
Update SubCusData Set IDNO = @New_IDNO, CusStatus = '02', Status = 'D', SubCanCelDate = Convert(Date, @CDate), ... Where BranchId = @Old_BranchId and
Account = @Old_Account;
新分公司則是 INSERT 一筆全新 row(New_BranchId + New_Account)。因為 BranchId 本身就變了,這個情境屬於「同 IDNO、不同 BranchId 不同 Account」,不算你問的「同
BranchId+IDNO」情況。

⚠️ 附帶發現:這個情境會讓 GetSubBaseData 查詢結果不穩定

情境 A 一旦發生,SubSendGW/Utils.cs:232-580(GetSubBaseData,前面查詢條件是 Where BranchId = @BranchId and IDNO = @IDNO——沒有濾Status,也沒有 ORDER BY,如果表裡同時有舊(D)跟新(6/7/8/9)兩筆同 IDNO+BranchId 的 row,這個 while (Rea迴圈(:598-809)會把兩筆都讀出來、後面的直接覆蓋前面的欄位值,最 回傳順序,沒有明確保證一定拿到「新的那筆」。這代表:如果情境 A真的發生過,且剛好客戶又在舊帳戶還沒被 UpdateNewIDNO 騰空 IDNO 之前重新開戶送出電文,GetSubBaseData 可能撈到錯的(舊、已銷戶)那筆資料去組電文。這是我認為值得放進你

