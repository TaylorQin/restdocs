# restdocs
maven+springboot+springfox+swagger2markup+spring restdoc+asciidoctor generate rest document
1.预备知识
建议了解swagger、Asciidoc、asciidoctor-maven-plugin和SpringBoot Testing。本人觉得Asciidoc官方介绍看起来比较晦涩，因此主要参考下面两篇内容：
使用Asciidoc代替Markdown和Word撰写开发文档
AsciidocFX Asciidoc的开源编辑器，连Markdown也支持。
2.关于Springfox和Spring Rest Docs
借用官网的一句话介绍Springfox：Automated JSON API documentation for API's built with Spring。我理解就是为基于Spring构建的API自动生成文档。
最终目标是为我们的项目生成如下形式的文档：





Springfox形式的文档

Spring Rest Docs：Document RESTful services by combining hand-written documentation with auto-generated snippets produced with Spring MVC Test. 本文主要用Spring Rest Docs来生成API的例子。
3.在线文档与离线文档
必须要说明，这里的目标是自动生成被官方称为staticdocs的文档（暂且理解为离线文档），对应Springfox官方文档的6. Configuring springfox-staticdocs。如果是在线文档，可以参考这篇博文：Spring Boot中使用Swagger2构建强大的RESTful API文档
4.从Maven依赖开始
在Spring Boot中使用Swagger2构建强大的RESTful API文档这篇博文中已经知道了如何生成在线文档，其实我们的思路就是把在线文档转成staticdocs形式的文档，因此该篇博文引用的依赖都要引入，Spring Rest Docs的依赖spring-restdocs-mockmvc，离线文档的依赖springfox-staticdocs，因为要在单元测试的时候生成文档，所以再加测试相关的spring-boot-starter-test。
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>2.6.1</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.6.1</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.restdocs</groupId>
            <artifactId>spring-restdocs-mockmvc</artifactId>
            <version>1.1.2.RELEASE</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-staticdocs</artifactId>
            <version>2.6.1</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.8</version>
        </dependency>
    </dependencies>

5.Maven插件
asciidoctor-maven-plugin插件是用来把Asciidoc格式转成HTML5格式。
<build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <includes>
                        <include>**/*Documentation.java</include>
                    </includes>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.asciidoctor</groupId>
                <artifactId>asciidoctor-maven-plugin</artifactId>
                <version>1.5.3</version>

                <!-- Configure generic document generation settings -->
                <configuration>
                    <sourceDirectory>${project.basedir}/docs/asciidoc</sourceDirectory>
                    <sourceDocumentName>index.adoc</sourceDocumentName>
                    <attributes>
                        <doctype>book</doctype>
                        <toc>left</toc>
                        <toclevels>3</toclevels>
                        <numbered></numbered>
                        <hardbreaks></hardbreaks>
                        <sectlinks></sectlinks>
                        <sectanchors></sectanchors>
                        <generated>${project.build.directory}/asciidoc</generated>
                    </attributes>
                </configuration>
                <!-- Since each execution can only handle one backend, run
                     separate executions for each desired output type -->
                <executions>
                    <execution>
                        <id>output-html</id>
                        <phase>test</phase>
                        <goals>
                            <goal>process-asciidoc</goal>
                        </goals>
                        <configuration>
                            <backend>html5</backend>
                            <!--<outputDirectory>${project.basedir}/docs/asciidoc/html</outputDirectory>-->
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

6.文档生成类的编写
先贴代码：
@AutoConfigureMockMvc
@AutoConfigureRestDocs(outputDir = "target/generated-snippets")
@RunWith(SpringRunner.class)
@SpringBootTest
public class Documentation {

    private String snippetDir = "target/generated-snippets";
    private String outputDir = "target/asciidoc";
    //private String indexDoc = "docs/asciidoc/index.adoc";

    @Autowired
    private MockMvc mockMvc;

    @After
    public void Test() throws Exception{
        // 得到swagger.json,写入outputDir目录中
        mockMvc.perform(get("/v2/api-docs").accept(MediaType.APPLICATION_JSON))
                .andDo(SwaggerResultHandler.outputDirectory(outputDir).build())
                .andExpect(status().isOk())
                .andReturn();

        // 读取上一步生成的swagger.json转成asciiDoc,写入到outputDir
        // 这个outputDir必须和插件里面<generated></generated>标签配置一致
        Swagger2MarkupConverter.from(outputDir + "/swagger.json")
                .withPathsGroupedBy(GroupBy.TAGS)// 按tag排序
                .withMarkupLanguage(MarkupLanguage.ASCIIDOC)// 格式
                .withExamples(snippetDir)
                .build()
                .intoFolder(outputDir);// 输出
    }

    @Test
    public void TestApi() throws Exception{
        mockMvc.perform(get("/student").param("name", "xxx")
                .accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andDo(document("getStudent", preprocessResponse(prettyPrint())));

        Student student = new Student();
        student.setName("xxx");
        student.setAge(23);
        student.setAddress("湖北麻城");
        student.setCls("二年级");
        student.setSex("男");

        mockMvc.perform(post("/student").contentType(MediaType.APPLICATION_JSON)
                .content(JSON.toJSONString(student))
                .accept(MediaType.APPLICATION_JSON))
                .andExpect(status().is2xxSuccessful())
                .andDo(document("addStudent", preprocessResponse(prettyPrint())));
    }
}

这个类包含两个方法，TestApi()是用来生成例子，Test()用来生成Asciidoc的文档。生成例子用到了spring-restdocs-mockmvc，每一个API都要进行单元测试才能生成相应的文档片段（snippets），生成的结果如图：






API使用例子片段

生成完整的Asciidoc文档用到了Swagger2MarkupConverter，第一步先获取在线版本的文档并保存到文件swagger.json中，第二步把swagger.json和之前的例子snippets整合并保存为Asciidoc格式的完整文档。生成结果如图：






Asciidoc格式的文档

7.Swagger的配置
这部分可以定义一些文档相关的信息。
@Configuration
@EnableSwagger2
public class SwaggerConfiguration {

    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
            .apiInfo(apiInfo())
            .select()
            .apis(RequestHandlerSelectors.basePackage("com.chinamobile.iot.controller"))
            .paths(PathSelectors.any())
            .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
            .title("Student info query api")
            .description("Some API to operator student information")
            .termsOfServiceUrl("http://iot.10086.com/")
            .version("1.0")
            .build();
    }
}

8.生成HTML5的文档
由于前面已经配置了maven的插件，只需要执行打包就可以生成相应的文档，如图：






最终文档

作者：quiterr
链接：https://www.jianshu.com/p/af7a6f29bf4f
来源：简书
简书著作权归作者所有，任何形式的转载都请联系作者获得授权并注明出处。
