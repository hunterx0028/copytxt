read me




// 原 DAO:87-127 / 598-637 的搜尋條件抽出，兩張表共用
// UnFinish 不放在這裡，改由呼叫端決定要查哪張表
// idnoParam：筆數用 "@IDNOS"、清單用 "@IDNO"，維持原本各自的行為
private static string BuildSubWhere(IASQRY002_QRYVO whereVo, string idnoParam)
{
    string w = "";
    //分公司
    if (!string.IsNullOrEmpty(whereVo.BranchId))
        w += (whereVo.BranchId != "ALL")
            ? @" AND (BranchId=@BranchId or OBranchId=@BranchId) "
            : @" AND (BranchId in " + whereVo.BranchIdList + " or OBranchId in " + whereVo.BranchIdList + ")";
    //帳號
    if (!string.IsNullOrEmpty(whereVo.SubAccount))    w += @" AND Account=@SubAccount ";
    //身份證號/統編
    if (!string.IsNullOrEmpty(whereVo.IDNO))          w += @" AND IDNO like " + idnoParam + " + '%' ";
    //客戶姓名
    if (!string.IsNullOrEmpty(whereVo.CustName))      w += @" AND CustName=@CustName ";
    //資料處理狀態
    if (!string.IsNullOrEmpty(whereVo.Status) && whereVo.Status != "ALL") w += @" AND Status=@Status ";
    //複委託沒有劃撥銀行帳號，故意塞此條件讓劃撥帳號查詢取不到複委託資料
    if (!string.IsNullOrEmpty(whereVo.DeliveryActNo)) w += @" AND Account=@DeliveryActNo ";
    // 資料登打日期
    if (!string.IsNullOrEmpty(whereVo.sCreateDate))   w += @" AND Convert(varchar(10),CreateDate,112) >= @SCreateDate ";
    if (!string.IsNullOrEmpty(whereVo.eCreateDate))   w += @" AND Convert(varchar(10),CreateDate,112) <= @ECreateDate ";
    return w;
}

筆數：GetDataCount，取代 :82-130

string subCntCols = @"select CreditSeqNo,BranchId,Account,IDNO,Mobile2,Status,KYCStatus,ActType ";
string subCntWhere = BuildSubWhere(whereVo, "@IDNOS");

sql += @"), SubData1 as ( ";
// 未完成件(Status 非 9/D)只會在 Transit，SubCusData 不用查
if (whereVo.UnFinish != "Y")
    sql += subCntCols + " from SubCusData with(nolock) where 1n all ";
sql += subCntCols + " from SubCusDataTransit with(nolock) where 1 = 1 " + subCntWhere;

清單：GetDataContent，取代 :592-641

string subCols = @"select CreditSeqNo, QRCodeSeqNo as OpenSourceSeqNo,BranchId,OBranchId,Account,IDNO,CustName,Status,'' as ActStatus,KYCStatus,ActType,''
as IdKind, isCopyFlag, PL_PkNo as OLStkSeqNo,
                          OpenSource, AddSubFlag, SubOpenDate ODate, SubCanCelDate CDate,
                          AddSubMemo=(case When BranchId in (''Y' then '8889' else null end) ";
string subWhere = BuildSubWhere(whereVo, "@IDNO");

sql += @"), SubData1 as ( ";
// 未完成件(Status 非 9/D)只會在 Transit，SubCusData 不用查
if (whereVo.UnFinish != "Y")
    sql += subCols + " from SubCusData with(nolock) where 1 = ";
sql += subCols + " from SubCusDataTransit with(nolock) where 1 = 1 " + subWhere;
