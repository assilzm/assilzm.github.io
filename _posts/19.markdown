---
author: admin
comments: true
date: 2016-12-06 13:46:49+00:00
layout: post
title: gollum增加unicode文件名支持
categories:
- gollum
---
## 文件路径
修改$gems/gitlab-grit-2.8.1/lib/grit/index.rb中write_tree函数，增加unicode解析支持

```ruby
def write_tree(tree = nil, now_tree = nil)
  tree = self.tree if !tree
  tree_contents = {}

  # fill in original tree
  now_tree = read_tree(now_tree) if(now_tree && now_tree.is_a?(String))
  now_tree.contents.each do |obj|
    sha = [obj.id].pack("H*")
    k = obj.name
    k += '/' if (obj.class == Grit::Tree)
    tmode = obj.mode.to_i.to_s  ## remove zero-padding
    tree_contents[k] = "%s %s\0%s" % [tmode, obj.name.force_encoding('ASCII-8BIT'), sha]      
  end if now_tree
  ```
