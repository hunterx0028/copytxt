read me




// 原 DAO:598-640 的搜尋條件抽出,兩張表共用
private static string BuildSubWhere(IASQRY002_QRYVO whereVo)
{
    string w = "";
    if (!string.IsNullOrEmpty(whereVo.BranchId))
        w += (whereVo.BranchId != "ALL")
            ? @" AND (BranchId=@BranchId or OBranchId=@BranchId) "
            : @" AND (BranchId in " + whereVo.BranchIdList + " or OBranchId in " + whereVo.BranchIdList + ")";
    if (!string.IsNullOrEmpty(whereVo.SubAccount))   w += @" AND Account=@SubAccount ";
    if (!string.IsNullOrEmpty(whereVo.IDNO))         w += @" AND IDNO like @IDNO + '%' ";
    if (!string.IsNullOrEmpty(whereVo.CustName))     w += @" AND CustName=@CustName ";
    if (!string.IsNullOrEmpty(whereVo.Status) && whereVo.Status != "ALL") w += @" AND Status=@Status ";
    if (!string.IsNullOrEmpty(whereVo.DeliveryActNo)) w += @" AND Account=@DeliveryActNo ";
    if (!string.IsNullOrEmpty(whereVo.sCreateDate))  w += @" AND Convert(varchar(10),CreateDate,112) >= @SCreateDate ";
    if (!string.IsNullOrEmpty(whereVo.eCreateDate))  w += @" AND Convert(varchar(10),CreateDate,112) <= @ECreateDate ";
    if (whereVo.UnFinish == "Y")                     w += @" and Status not in ('9','D') ";
    return w;
}

// GetDataContent 內,取代原本的 SubData1
string subCols = @"select CreditSeqNo, QRCodeSeqNo as OpenSourceSeqNo, BranchId, OBranchId, Account, IDNO, CustName, Status,
                          '' as ActStatus, KYCStatus, ActType, '' as IdKind, isCopyFlag, PL_PkNo as OLStkSeqNo,
                          OpenSource, AddSubFlag, SubOpenDate ODate, SubCanCelDate CDate,
                          AddSubMemo=(case When BranchId in ('888L','888M') and AddSubFlag='Y' then '8889' else null end) ";
string subWhere = BuildSubWhere(whereVo);
sql += @"), SubData1 as ( "
     + subCols + " from SubCusData where 1=1 " + subWhere
     + " union all "
     + subCols + " from SubCusDataTransit where 1=1 " + subWhere;
