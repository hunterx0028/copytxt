read me

string tdWhere = " where BranchId = '" + BranchId + "' And IDNO like '" + IDNO + "'+'%' and Status<>'D' and (Convert(varchar, CreateDate, 112)=Convert(varchar, GETDATE(), 112) or Convert(varchar, ModifiedDate, 112)=Convert(varchar, GETDATE(), 112))";
sql += "select IDNO, ActType from SubCusData " + tdWhere
     + " union all "
     + "select IDNO, ActType from SubCusDataTransit " + tdWhere;

