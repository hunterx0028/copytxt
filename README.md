read me




// GetDataContent 內，取代原本的 SubData1
string subWhere = BuildSubWhere(whereVo);
sql += @"), SubData1 as ( ";
// 未完成件(Status 非 9/D)只會在 Transit，SubCusData 不用查
if (whereVo.UnFinish != "Y")
    sql += subCols + " from SubCusData where 1=1 " + subWhere + " union all ";
sql += subCols + " from SubCusDataTransit where 1=1 " + subWhere;

subCols 跟上一版一樣，沒有改。
