read me

sql += @"select IDNO, PL_PkNo, Status, ActType from SubCusData where PL_PkNo=@PL_PkNo and idno like '" + IDNO + "%' "
     + @" union all "
     + @" select IDNO, PL_PkNo, Status, ActType from SubCusDataTransit where PL_PkNo=@PL_PkNo and idno like '" + IDNO + "%' "
     + @" order by IDNO";

