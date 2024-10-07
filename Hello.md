Here’s a basic structure and code layout for the Maven project using TestNG, Excel-based data, and Log4j logging. This will help automate the login, add-to-cart, checkout, and logout processes.

1. Project Structure

my-ecommerce-test
├── src
│   ├── main
│   │   └── java
│   │       └── com.yourcompany.base
│   │           └── BaseTest.java
│   ├── test
│   │   └── java
│   │       └── com.yourcompany.testpackage
│   │           ├── LoginTest.java
│   │           ├── AddToCartTest.java
│   │           ├── CheckoutTest.java
│   │           ├── LogoutTest.java
│   │           └── EndToEndTest.java
│   └── resources
│       ├── testng.xml
│       └── log4j2.xml
└── pom.xml

2. pom.xml

Add dependencies for TestNG, Apache POI (for Excel), and Log4j:

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.yourcompany</groupId>
    <artifactId>my-ecommerce-test</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <!-- TestNG -->
        <dependency>
            <groupId>org.testng</groupId>
            <artifactId>testng</artifactId>
            <version>7.4.0</version>
            <scope>test</scope>
        </dependency>

        <!-- Apache POI for Excel -->
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi</artifactId>
            <version>5.0.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-ooxml</artifactId>
            <version>5.0.0</version>
        </dependency>

        <!-- Log4j -->
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.14.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>2.14.0</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>2.22.2</version>
            </plugin>
        </plugins>
    </build>
</project>

3. BaseTest.java

This will initialize the WebDriver and handle setup/teardown operations.

package com.yourcompany.base;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.testng.annotations.AfterClass;
import org.testng.annotations.BeforeClass;

public class BaseTest {
    protected WebDriver driver;

    @BeforeClass
    public void setup() {
        System.setProperty("webdriver.chrome.driver", "path/to/chromedriver");
        driver = new ChromeDriver();
        driver.manage().window().maximize();
    }

    @AfterClass
    public void teardown() {
        if (driver != null) {
            driver.quit();
        }
    }
}

4. ExcelUtil.java

Utility class for reading data from Excel.

package com.yourcompany.utils;

import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;

import java.io.FileInputStream;
import java.io.IOException;

public class ExcelUtil {
    private Workbook workbook;

    public ExcelUtil(String excelFilePath) throws IOException {
        FileInputStream fis = new FileInputStream(excelFilePath);
        workbook = new XSSFWorkbook(fis);
    }

    public String getData(int sheetNumber, int row, int col) {
        Sheet sheet = workbook.getSheetAt(sheetNumber);
        Row rowData = sheet.getRow(row);
        return rowData.getCell(col).getStringCellValue();
    }

    public int getRowCount(int sheetNumber) {
        return workbook.getSheetAt(sheetNumber).getLastRowNum() + 1;
    }
}

5. Test Classes (LoginTest.java, AddToCartTest.java, CheckoutTest.java, and LogoutTest.java)

Each class will inherit from BaseTest and perform specific actions. Here’s an example for the LoginTest:

package com.yourcompany.testpackage;

import com.yourcompany.base.BaseTest;
import com.yourcompany.utils.ExcelUtil;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.testng.annotations.BeforeClass;
import org.testng.annotations.Test;

import java.io.IOException;

public class LoginTest extends BaseTest {
    private ExcelUtil excelUtil;
    private Logger logger = LogManager.getLogger(LoginTest.class);

    @BeforeClass
    public void setupTest() throws IOException {
        excelUtil = new ExcelUtil("path/to/testdata.xlsx");
    }

    @Test
    public void loginTest() {
        for (int i = 1; i < excelUtil.getRowCount(0); i++) {
            String username = excelUtil.getData(0, i, 0);
            String password = excelUtil.getData(0, i, 1);

            // Perform login operations here using driver, e.g., driver.findElement(...) 
            logger.info("Logging in with user: " + username);

            // Placeholder for login implementation
        }
    }
}

Repeat similar structures for AddToCartTest, CheckoutTest, and LogoutTest, with appropriate methods for each step.

6. EndToEndTest.java

This will chain all the individual test classes together:

package com.yourcompany.testpackage;

import org.testng.annotations.Test;

public class EndToEndTest {
    @Test
    public void endToEndTest() {
        LoginTest login = new LoginTest();
        login.loginTest();

        AddToCartTest addToCart = new AddToCartTest();
        addToCart.addToCartTest();

        CheckoutTest checkout = new CheckoutTest();
        checkout.checkoutTest();

        LogoutTest logout = new LogoutTest();
        logout.logoutTest();
    }
}

7. log4j2.xml

Configure Log4j2 for logging:

<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d [%t] %-5level: %msg%n%throwable"/>
        </Console>
        <File name="File" fileName="logs/testlog.log">
            <PatternLayout pattern="%d [%t] %-5level: %msg%n%throwable"/>
        </File>
    </Appenders>
    <Loggers>
        <Root level="info">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="File"/>
        </Root>
    </Loggers>
</Configuration>

8. testng.xml

Define the test suite and sequence:

<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="EcommerceTestSuite">
    <test name="EndToEndTests">
        <classes>
            <class name="com.yourcompany.testpackage.EndToEndTest"/>
        </classes>
    </test>
</suite>

9. Running the Tests

Use the following Maven command to run the tests:

mvn test -Dsurefire.suiteXmlFiles=src/resources/testng.xml


This structure and code should help you implement the login, add-to-cart, checkout, and logout automation with data-driven testing from Excel. Let me know if you'd like to expand or adjust any part of it!

