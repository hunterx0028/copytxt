read me

#: A
位置: GetDataCount 的 SubData1(DAO:83-130)和 GetDataContent 的 SubData1(DAO:592-641)
要改什麼: UNION ALL SubCusDataTransit。建議把搜尋條件抽成一個共用方法,兩張表、兩個查詢共四處都套用同一份。這樣也能順便修掉 GetDataCount 用字串拼接 IDNOS
的問題(兩邊條件目前不一致)
────────────────────────────────────────
#: B
位置: AfterReDoCredit_UpdateDate.ashx(sub 分支)和 DoUpdateDateSub(DAO:1263-1320)
要改什麼: 這個動作只會發生在 Status 6/E1,資料一定在 Transit。GetSingleSubCusData 和四個 UPDATE 改成對應的 *Transit 表,WHERE 可以維持 BranchId + IDNO
────────────────────────────────────────
#: C
位置: IASQRY016DAO.GetPL_OLAcctOpenData(:1102)、SubCusDataDAO.CheckPLOpenData(:1095)、SubCusDataDAO.CheckTDayOpenDataCnt(:1143)
要改什麼: 改成兩表都查,理由同上一則回覆。這三支是共用 DAO,要評估對 IASCUS013 的影響



查詢筆數及結果資料表

修改 1:GetDataCount(DAO:83-130)

                    sql += @"), SubData1 as
                        (
                                select SeqNo,CreditSeqNo,BranchId,Account,IDNO,CustName,Tel,CompTel,Mobile1,Mobile2,Status,KYCStatus,ActType from SubCusData with(nolock) where 1 = 1  ";

                    #region <搜尋條件>
                    // ... 原 87-129 行,完全不動 ...
                    #endregion

                    // 新增:未完成送電文的複委託資料
                    sql += @" UNION ALL
                                select SeqNo,CreditSeqNo,BranchId,Account,IDNO,CustName,Tel,CompTel,Mobile1,Mobile2,Status,KYCStatus,ActType from SubCusDataTransit with(nolock) where 1 = 1  ";

                    #region <搜尋條件>
                    // ... 把原 87-129 行整段複製到這裡 ...
                    #endregion

                    sql += @"), TrsData1 as   // ← 原本的下一段,不動

修改 2:GetDataContent(DAO:592-641)

                    sql += @"), SubData1 as
                        (
                                select SeqNo, CreditSeqNo, QRCodeSeqNo as OpenSourceSeqNo,BranchId,OBranchId,Account,IDNO,CustName,Status,'' as 
ActStatus,KYCStatus,ActType,'' as IdKind, isCopyFlag, PL_PkNo
                                                        OpenSource, AddSubFlag, SubOpenDate ODate, SubCanCelDate CDate,
                                                        AddSub'888L','888M') and AddSubFlag='Y' then '8889' else null end)--暫時的
                            from SubCusData with(nolock) where
                    #region <搜尋條件>
                    // ... 原 598-640 行,完全不動 ...
                    #endregion

                    // 新增:未完成送電文的複委託資料
                    sql += @" UNION ALL
                                select SeqNo, CreditSeqNo, QRCodeSeqNo as OpenSourceSeqNo,BranchId,OBranchId,Account,IDNO,CustName,Status,'' as
ActStatus,KYCStatus,ActType,'' as IdKind, isCopyFlag, PL_PkNo
                                                        OpenSource, AddSubFlag, SubOpenDate ODate, SubCanCelDate CDate,
                                                        AddSub'888L','888M') and AddSubFlag='Y' then '8889' else null end)--暫時的
                            from SubCusDataTransit with(nolock
                    #region <搜尋條件>
                    // ... 把原 598-640 行整段複製到這裡 ...
                    #endregion

                    sql += @"), TrsData1 as   // ← 原本的下一段,不動
					
重做徵信時 1291
