<a name="kRORf"></a>
# 解析excel表导出到SQLite 数据库
```python
#!/usr/bin/python
#coding:utf-8
import sqlite3
import xlrd
import csv
import importlib
import sys
import os

if sys.version_info[0] == 3:
    importlib.reload(sys)
else:
    reload(sys)
    sys.setdefaultencoding('utf8')

# 写入数据到 csv 文件的全局容器
totalCsvArr = []

def open_excel(file):
    try:
        data = xlrd.open_workbook(file)
        return data
    except Exception as ex:
        print(str(ex))

def excel_table_byname(excelfile, by_name):
    data = excelfile
    table = data.sheet_by_name(by_name)
    return table
 
def excel_table_byIdx(excelfile, by_idx):
    data = excelfile
    table = data.sheet_by_index(by_idx)
    return table

def excel_sheet_byColIdx(sheetName, colIdx):
    table = sheetName
    cols =  table.col_values(colIdx) #某一列数据
    return cols
 
def excel_sheet_byRowIdx(sheetName, rowIdx):
    table = sheetName
    rows =  table.row_values(rowIdx) #某一行数据
    return rows

# 校验字符串是否是中文字符串
def check_str_for_chinese(strs, tableName, fieldName, sn):
    is_contains = False
    result_str = strs
    if isinstance(strs, str):
        for _char in strs:
            if _char >= u'\u4e00' and _char <= u'\u9fa5':
                is_contains = True
                break
        # 修改数据库数据 并将原数据写入 csv
        if is_contains:
            result_str = "%s_%s_%s" % (tableName, fieldName, int(sn))
            tempArr = []
            tempArr.append(result_str)
            tempArr.append('"%s"' % strs)
            totalCsvArr.append(tempArr)
    return result_str


# 根据rowIdx筛选第0行包含"L"并且第2行不为空的数据
# 去除#开头的行
def excel_sheet_byRow_filter(sheetName, tableName, rowIdx):
    rows_0 = excel_sheet_byRowIdx(sheetName, 0)     # 获得第0行数据
    rows_2 = excel_sheet_byRowIdx(sheetName, 2)     # 获得第2行数据
    rows = excel_sheet_byRowIdx(sheetName, rowIdx)  # 获得rowIdx行数据
    list =[]
    for rownum in range(len(rows)):
        rowVlaue_0 = -1
        if rownum < len(rows_0):
            rowVlaue_0 = rows_0[rownum]
        rowValue_2 = -1
        if rownum < len(rows_2):
            rowValue_2 = rows_2[rownum]
        rowValue = rows[rownum]
        if (rowVlaue_0 != -1) and (rowVlaue_0.find("L") != -1) and (rowValue_2 != -1) and (rowValue_2 != ""):
            # 默认sn 都在第一列
            if (len(list) > 0 and rowIdx >= 4):
                list.append(check_str_for_chinese(rowValue, tableName, rowValue_2, list[0]))
            else:
                list.append(rowValue)
        if (rownum == 0) and (str(rowValue).startswith("#")):
            list = []
            break
    return list

# 生成一张表
def run_excel_to_sqlite(dbPath, excelPath):
    # 定义数据库实例对象
    execObj = ExcelToSqlite(dbPath)
    # 获取excel表sheets，然后分别处理每个sheet
    curExcel = open_excel(excelPath)
    sheetCount = len(curExcel.sheets())
    sheetNames = curExcel.sheet_names()
    for sheetIndex in range(sheetCount):
        print(sheetIndex)
        curSheet = curExcel.sheets()[sheetIndex]
        curSheetName = sheetNames[sheetIndex]
        if curSheet and (curSheetName.find("|") != -1):
            # 根据sheet名获得表名
            tableName = curSheetName.split("|")[1]
            print(tableName)
            # 将sheet转为db
            totalCsvArr = []
            execObj.ExcelToDb(curSheet, tableName)
            # 写入 csv
            with open(tableName + ".csv", "w") as csvfile:
                writer = csv.writer(csvfile, lineterminator='\n',quotechar='', quoting=csv.QUOTE_NONE)
                # first write columns_name
                writer.writerow(["Key", "SourceString"])
                # then write data
                writer.writerows(totalCsvArr)


class ExcelToSqlite(object):
    exe = u"     执行: "
    output = u"     输出: "
    sheetDataStartIndex = 4  # 数据开始计算的行数，如第0行是表头，第1行及之后是数据
 
    def __init__(self, dbName):
        print(u"初始化数据库实例")
        super(ExcelToSqlite, self).__init__()
        self.conn = sqlite3.connect(dbName)
        self.cursor = self.conn.cursor()
 
    def __del__(self):
        print(u"释放数据库实例")
        self.cursor.close()
        self.conn.close()
 
    def ExcelToDb(self, sheetTable, tableName):
        """
        excel转化为sqlite数据库表
        :param sheetTable:当前处理的sheet
        :param tableName:数据库表名
        """
        print(u"Excel文件 转 db")
        self.tableName = tableName
        self.sheetRows = sheetTable.nrows  # excel 行数
        self.sheetCols = sheetTable.ncols  # excle 列数
        print(self.sheetRows, self.sheetCols)
        fieldNames = excel_sheet_byRow_filter(sheetTable, tableName, 2)  # 得到表头字段名
        # 创建表
        fieldTypes = ""
        for index in range(len(fieldNames)):
            if (index != len(fieldNames) - 1):
                fieldTypes += '[%s] text,' % fieldNames[index]
            else:
                fieldTypes += '[%s] text' % fieldNames[index]
        if fieldTypes == "":
            return
        self.__CreateTable(tableName, fieldTypes)
        # 插入数据类型数据
        fieldTypes = excel_sheet_byRow_filter(sheetTable, tableName, 1)  # 得到表头字段名
        if len(fieldTypes) != 0:
            self.__Insert(fieldNames, fieldTypes)
        # 插入数据
        for rowId in range(self.sheetDataStartIndex, self.sheetRows):
            fieldValues = excel_sheet_byRow_filter(sheetTable, tableName, rowId)
            if len(fieldValues) == 0:
                continue
            self.__Insert(fieldNames, fieldValues)
 
    def __CreateTable(self, tableName, field):
        """
        创建表
        :param tableName: 表名
        :param field: 字段名及类型
        :return:
        """
        print(u"创建表 " + tableName)
        sql = 'create table if not exists %s(%s)' % (self.tableName, field)  # primary key not null
        print(self.exe + sql)
        self.cursor.execute(sql)
        self.conn.commit()

        # 默认在执行建表后清空一次数据表防止重复导入数据
        print('清空表:' + self.tableName)
        sql_clear_table = 'delete from %s' % self.tableName
        self.cursor.execute(sql_clear_table)
        self.conn.commit()
 
    def __Insert(self, fieldNames, fieldValues):
        """
        插入数据
        :param fieldNames: 字段list
        :param fieldValues: 值list
        """
        # 通过fieldNames解析出字段名
        names = ""  # 字段名，用于插入数据
        nameTypes = ""  # 字段名及字段类型，用于创建表
        for index in range(len(fieldNames)):
            if (index != len(fieldNames) - 1):
                names += '[%s],' % fieldNames[index]
                nameTypes += '%s text,' % fieldNames[index]
            else:
                names += '[%s]' % fieldNames[index]
                nameTypes += '%s text,' % fieldNames[index]
        # 通过fieldValues解析出字段对应的值
        values = ""
        for index in range(len(fieldValues)):
            cell_value = str((fieldValues[index]))
            if (isinstance(fieldValues[index], float) and fieldValues[index] % 1 == 0):
                cell_value = str((int)(fieldValues[index]))  # 读取的excel数据会自动变为浮点型，这里转化为文本
            if (index != len(fieldValues) - 1):
                values += "\'" + cell_value + "\',"
            else:
                values += "\'" + cell_value + "\'"
        values = values.replace(u'\xa0', u' ')
        # 将fieldValues解析出的值插入数据库
        sql = 'insert into %s(%s) values(%s)' % (self.tableName, names, values)
        print(self.exe + sql)
        self.cursor.execute(sql)
        self.conn.commit()
 
    def Query(self, tableName):
        """
        查询数据库表中的数据
        :param tableName:表名
        """
        print(u"查询表 " + tableName)
        sql = 'select * from %s' % (tableName)
        print(self.exe + sql)
        self.cursor.execute(sql)
        results = self.cursor.fetchall()  # 获取所有记录列表
        index = 0
        for row in results:
            print(self.output + "index=" + index.__str__() + " detail=" + str(row))  # 打印结果
            index += 1
        print(self.output + u"共计" + results.__len__().__str__() + u"条数据")
 
    def executeSqlCommand(self, sqlCommand):
        """
        执行输入的sql命令
        :param sqlCommand: sql命令
        """
        print(u"执行自定义sql " + tableName)
        print(self.exe + sqlCommand)
        self.cursor.execute(sqlCommand)
        results = self.cursor.fetchall()
        print(self.output + str(results))
        for index in range(0, results.__len__()):
            print(self.output + str(results[index]))
        self.conn.commit()
 
if __name__ == '__main__':
    configPath = os.path.join(os.path.dirname(__file__), os.path.pardir)
    resourcePath = os.path.abspath(os.path.join(configPath, os.path.pardir, 'resource'))
    if os.path.exists(resourcePath):
        # 数据库路径
        dbName = os.path.join(resourcePath, 'Content/Datas/SQLite/Main.db')
        # excel路径
        FileList = os.listdir(configPath)
        for file in FileList:
            if file.endswith('.xlsx'):
                run_excel_to_sqlite(dbName, os.path.join(configPath, file))
    else:
        print(f'Path not found: {resourcePath}')

```
