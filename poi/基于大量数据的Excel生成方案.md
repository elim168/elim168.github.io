# 基于大量数据的Excel生成方案

以往我们在基于POI生成Excel文件时，都是利用官方提供的HSSF或XSSF对应的系列API，它们操作简便，上手比较快。但是对于大数据量的Excel文件生成往往会比较耗时，这是我们利用标准的API进行开发的一个痛点。对于性能更高一点的API，POI官方会建议我们使用SXSSF系列API，虽然它的性能比起HSSF和XSSF会有很大的提高，但是面对大量数据的时候还是会比较慢，为此官方还给我们提供了一种基于XML的方案。  

其实对于一个Excel文件来说，最核心的是它的数据。Excel文件中的数据和样式文件是分开存储的，它们都对应于它自己体系中的一个XML文件。有兴趣的朋友可以把Excel文件的后缀名改成“`.zip`”，然后用压缩文件把它解压缩，可以看到它里面的结构是由一堆的XML文件组成的。如果我们把解压缩后的文件再压缩成一个压缩文件，并把它的后缀名改为Excel文件对应的后缀名“`.xlsx`”或“`.xls`”，然后再用Excel程序把它打开。这个时候你会发现它也是可以打开的。笔者本文所要讲述的基于大量的数据生成Excel的方案就是基于这种XML文件的方案，它依赖于一个现有的Excel文件（这个Excel文件可以在运行时生成好），然后把我们的数据生成对应的XML表示，再把我们的XML替换原来的XML文件，再进行打包后就变成了一个Excel文件了。基于这种方式，笔者做了一个测试，生成了一个拥有3500万行，5列的Excel文件，该文件大小为1GB，耗时412秒。这种效率比起我们应用传统的API来说是指数倍的。  

细节的实现详情，请读者自己参考以下示例代码，该示例代码是笔者从Apache官方下载的，原地址是[https://svn.apache.org/repos/asf/poi/trunk/src/examples/src/org/apache/poi/xssf/usermodel/examples/BigGridDemo.java](https://svn.apache.org/repos/asf/poi/trunk/src/examples/src/org/apache/poi/xssf/usermodel/examples/BigGridDemo.java)。需要注意的是生成的XML中需要应用到的样式需要事先生成，需要应用函数、合并单元格等逻辑的时候，可以先拿一个Excel文件应用对应的函数、合并逻辑，再把它解压缩后查看里面的XML文件的展现形式，然后自己拼接的时候也拼接成对应的形式，这样自己生成的Excel文件也会有对应的效果。

```java
public class BigDataTest {

    private static final String XML_ENCODING = "UTF-8";
    
    public static void main(String[] args) throws Exception {

    	long start = System.currentTimeMillis();
    	
        // Step 1. Create a template file. Setup sheets and workbook-level objects such as
        // cell styles, number formats, etc.

        XSSFWorkbook wb = new XSSFWorkbook();
        XSSFSheet sheet = wb.createSheet("Big Grid");

        Map<String, XSSFCellStyle> styles = createStyles(wb);
        //name of the zip entry holding sheet data, e.g. /xl/worksheets/sheet1.xml
        String sheetRef = sheet.getPackagePart().getPartName().getName();

        //save the template
        FileOutputStream os = new FileOutputStream("template.xlsx");
        wb.write(os);
        os.close();

        //Step 2. Generate XML file.
        File tmp = File.createTempFile("sheet", ".xml");
        Writer fw = new OutputStreamWriter(new FileOutputStream(tmp), XML_ENCODING);
        generate(fw, styles);
        fw.close();

        //Step 3. Substitute the template entry with the generated data
        FileOutputStream out = new FileOutputStream("D:/big-grid2.xlsx");
        //用心拼接生成的XML文件替换原来模板Excel文件中对应的XML文件，再压缩打包为一个Excel文件。
        substitute(new File("template.xlsx"), tmp, sheetRef.substring(1), out);
        out.close();
        
        wb.close();
        
        long end = System.currentTimeMillis();
        
        System.out.println("耗时： " + (end - start));
    }

    /**
     * Create a library of cell styles.
     */
    private static Map<String, XSSFCellStyle> createStyles(XSSFWorkbook wb){
        Map<String, XSSFCellStyle> styles = new HashMap<String, XSSFCellStyle>();
        XSSFDataFormat fmt = wb.createDataFormat();

        XSSFCellStyle style1 = wb.createCellStyle();
        style1.setAlignment(HorizontalAlignment.RIGHT);
        style1.setDataFormat(fmt.getFormat("0.0%"));
        styles.put("percent", style1);

        XSSFCellStyle style2 = wb.createCellStyle();
        style2.setAlignment(HorizontalAlignment.CENTER);
        style2.setDataFormat(fmt.getFormat("0.0X"));
        styles.put("coeff", style2);

        XSSFCellStyle style3 = wb.createCellStyle();
        style3.setAlignment(HorizontalAlignment.RIGHT);
        style3.setDataFormat(fmt.getFormat("$#,##0.00"));
        styles.put("currency", style3);

        XSSFCellStyle style4 = wb.createCellStyle();
        style4.setAlignment(HorizontalAlignment.RIGHT);
        style4.setDataFormat(fmt.getFormat("mmm dd"));
        styles.put("date", style4);

        XSSFCellStyle style5 = wb.createCellStyle();
        XSSFFont headerFont = wb.createFont();
        headerFont.setBold(true);
        style5.setFillForegroundColor(IndexedColors.GREY_25_PERCENT.getIndex());
        style5.setFillPattern(FillPatternType.SOLID_FOREGROUND);
        style5.setFont(headerFont);
        styles.put("header", style5);

        return styles;
    }

    private static void generate(Writer out, Map<String, XSSFCellStyle> styles) throws Exception {

        Random rnd = new Random();
        Calendar calendar = Calendar.getInstance();

        SpreadsheetWriter sw = new SpreadsheetWriter(out);
        sw.beginSheet();

        //insert header row
        sw.insertRow(0);
        int styleIndex = styles.get("header").getIndex();
        sw.createCell(0, "Title", styleIndex);
        sw.createCell(1, "% Change", styleIndex);
        sw.createCell(2, "Ratio", styleIndex);
        sw.createCell(3, "Expenses", styleIndex);
        sw.createCell(4, "Date", styleIndex);

        sw.endRow();

        //write data rows
        for (int rownum = 1; rownum < 100; rownum++) {
            sw.insertRow(rownum);

            sw.createCell(0, "Hello, " + rownum + "!");
            sw.createCell(1, (double)rnd.nextInt(100)/100, styles.get("percent").getIndex());
            sw.createCell(2, (double)rnd.nextInt(10)/10, styles.get("coeff").getIndex());
            sw.createCell(3, rnd.nextInt(10000), styles.get("currency").getIndex());
            sw.createCell(4, calendar, styles.get("date").getIndex());

            sw.endRow();

            calendar.roll(Calendar.DAY_OF_YEAR, 1);
        }
        sw.endSheet();
    }

    /**
     *
     * @param zipfile the template file
     * @param tmpfile the XML file with the sheet data
     * @param entry the name of the sheet entry to substitute, e.g. xl/worksheets/sheet1.xml
     * @param out the stream to write the result to
     */
    private static void substitute(File zipfile, File tmpfile, String entry, OutputStream out) throws IOException {
        ZipFile zip = ZipHelper.openZipFile(zipfile);
        try {
            ZipOutputStream zos = new ZipOutputStream(out);
    
            Enumeration<? extends ZipEntry> en = zip.entries();
            while (en.hasMoreElements()) {
                ZipEntry ze = en.nextElement();
                if(!ze.getName().equals(entry)){
                    zos.putNextEntry(new ZipEntry(ze.getName()));
                    InputStream is = zip.getInputStream(ze);
                    copyStream(is, zos);
                    is.close();
                }
            }
            zos.putNextEntry(new ZipEntry(entry));
            InputStream is = new FileInputStream(tmpfile);
            copyStream(is, zos);
            is.close();
    
            zos.close();
        } finally {
            zip.close();
        }
    }

    private static void copyStream(InputStream in, OutputStream out) throws IOException {
        byte[] chunk = new byte[1024];
        int count;
        while ((count = in.read(chunk)) >=0 ) {
          out.write(chunk,0,count);
        }
    }

    /**
     * Writes spreadsheet data in a Writer.
     * (YK: in future it may evolve in a full-featured API for streaming data in Excel)
     */
    public static class SpreadsheetWriter {
        private final Writer _out;
        private int _rownum;

        public SpreadsheetWriter(Writer out){
            _out = out;
        }

        public void beginSheet() throws IOException {
            _out.write("<?xml version=\"1.0\" encoding=\""+XML_ENCODING+"\"?>" +
                    "<worksheet xmlns=\"http://schemas.openxmlformats.org/spreadsheetml/2006/main\">" );
            _out.write("<sheetData>\n");
        }

        public void endSheet() throws IOException {
            _out.write("</sheetData>");
            _out.write("</worksheet>");
        }

        /**
         * Insert a new row
         *
         * @param rownum 0-based row number
         */
        public void insertRow(int rownum) throws IOException {
            _out.write("<row r=\""+(rownum+1)+"\">\n");
            this._rownum = rownum;
        }

        /**
         * Insert row end marker
         */
        public void endRow() throws IOException {
            _out.write("</row>\n");
        }

        public void createCell(int columnIndex, String value, int styleIndex) throws IOException {
            String ref = new CellReference(_rownum, columnIndex).formatAsString();
            _out.write("<c r=\""+ref+"\" t=\"inlineStr\"");
            if(styleIndex != -1) _out.write(" s=\""+styleIndex+"\"");
            _out.write(">");
            _out.write("<is><t>"+value+"</t></is>");
            _out.write("</c>");
        }

        public void createCell(int columnIndex, String value) throws IOException {
            createCell(columnIndex, value, -1);
        }

        public void createCell(int columnIndex, double value, int styleIndex) throws IOException {
            String ref = new CellReference(_rownum, columnIndex).formatAsString();
            _out.write("<c r=\""+ref+"\" t=\"n\"");
            if(styleIndex != -1) _out.write(" s=\""+styleIndex+"\"");
            _out.write(">");
            _out.write("<v>"+value+"</v>");
            _out.write("</c>");
        }

        public void createCell(int columnIndex, double value) throws IOException {
            createCell(columnIndex, value, -1);
        }

        public void createCell(int columnIndex, Calendar value, int styleIndex) throws IOException {
            createCell(columnIndex, DateUtil.getExcelDate(value, false), styleIndex);
        }
    }
	
}
```

（注：本文由Elim写于2017年6月5日）