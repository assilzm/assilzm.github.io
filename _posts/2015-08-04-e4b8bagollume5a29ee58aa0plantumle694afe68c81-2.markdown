


之前为gollum增加了graphviz支持，后来考虑了下其实plantuml支持的流程格式更多，并且完全兼容graphviz。因此顺便也把plantuml的支持增加上了。







做完后发现其实增加plantuml的支持更简单些。















示例的gollum版本为4.0.0，操作系统为centos







gollum直接使用gem安装，gems目录路径使用$gems代替，该路径通常在ruby安装目录下。














##1、下载一份plantuml的[jar包](http://ncu.dl.sourceforge.net/project/plantuml/plantuml.jar)。







可能需要翻墙。下载完成后放到一个目录下。如：/root/plantuml/plantuml.jar。














##2、为_**gollum-lib**_增加一个plantuml的filter







新建文件：_**$gems/gollum-lib-4.0.X/lib/gollum-lib/filter/plantuml.rb**_







内容如下：





```ruby
    # ~*~ encoding: utf-8 ~*~
    require 'net/http'
    require 'uri'
    require 'open-uri'
    require File.expand_path '../../helpers', __FILE__

    # PlantUML Diagrams
    #
    # Render an inline plantuml diagram by generating a PNG image using the
    # plantuml.jar tool.
    #
    class Gollum::Filter::PlantUML < Gollum::Filter

      #path of plantuml.jar
      JAR = "/root/plantuml/plantuml.jar"
      #path of java
      JAVA= "java"

      # Extract all sequence diagram blocks into the map and replace with
      # placeholders.
      def extract(data)
        return data if @markup.format == :txt
        data.gsub(/(@startuml[\s\S]*?@enduml)/m) do
          id       = Digest::SHA1.hexdigest($1)
          @map[id] = { :code => $1 }
          id
        end
      end

      # Process all diagrams from the map and replace the placeholders with
      # the final HTML.
      #
      def process(data)
        out_path_dir = ::File.expand_path ::File.join(@markup.page.wiki.path, 'tmp')
        Dir.mkdir out_path_dir unless ::File.exists? out_path_dir
        @map.each do |id, spec|
          data.gsub!(id) do
            render_plantuml(id, spec[:code],out_path_dir)
          end
        end
        data
      end

      private

      def render_plantuml(id, code,filepath)
        out_path = ::File.join(filepath, id)
        unless File::exists?(filepath+"/"+id)
          File.open(filepath+"/"+id, "w") do |file|
                file << code
          end
        end
        unless File::exists?( filepath+"/"+id+".png" )
          puts("#{JAVA} -jar #{JAR} #{filepath}/#{id} -o '#{filepath}'")
          system("#{JAVA} -jar #{JAR} #{filepath}/#{id} -o '#{filepath}'")
            unless $?.success?
              html_error("failed to generate uml image")
            end
        end
        "<img src=\"tmp/#{id}.png\" />"
      end

    end
```













##3、让wiki格式化内容时使用该filter







修改_**$gems/gollum-lib-4.0.X/lib/gollum-lib/wiki.rb**_,为属性**_@filter_chain_**增加_**PlantUML**_对象，修改后的该属性为：
```ruby
@filter_chain         = options.fetch :filter_chain,[:Metadata, :PlainText, :TOC, :RemoteCode, :Code, :Macro, :Sanitize, :WSD, :Tags,:PlantUML, :Render]
```
OK，搞定。重启gollum，现在新建一个page，录入以下内容：
```ruby
    @startuml
    digraph G {
        main -> parse -> execute;
        main -> init;
        main -> cleanup;
        execute -> make_string;
        execute -> printf;
        init -> make_string;
        main -> printf;
        execute -> compare;
    }
@enduml
```





预览或保存，就可以看到流程图了。







![](http://www.assilzm.com/wp-content/uploads/2015/08/20150804175632.jpg)







生成的流程图文件会存放在$wikidir/wiki/tmp下
