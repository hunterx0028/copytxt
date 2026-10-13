read me

string subWhere = " where IDNO like '" + IDNO.Trim() + "' + '%'";
if (!string.IsNullOrEmpty(Pk_No)){
    subWhere += @" and PL_PkNo=@Pk_No ";
}
sql += @" )select M.*, S.IDNO, S.ActType, S.Status from Main M inner join ( "
     + " select BranchId, IDNO, ActType, Status from SubCusData " + subWhere
     + " union all "
     + " select BranchId, IDNO, ActType, Status from SubCusDataTransit " + subWhere
     + @" ) S on M.BranchId = S.BranchId and left(M.RIDNO,10) = left(S.IDNO,10)";
