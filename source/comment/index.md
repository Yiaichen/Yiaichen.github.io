---
title: 留言板
date: 2017-08-15 21:38:01
---
<div id="container"></div>
<link rel="stylesheet" href="https://jjeejj.github.io/css/gitment.css">
<script src="https://jjeejj.github.io/js/gitment.js"></script>
<script>
var gitment = new Gitment({
  id: 'window.location.pathname',
  owner: 'Yiaichen',  //改你自己的名字
  repo: 'Yiaichen.github.io',  //专门储存评论一个GitHub仓库
  oauth: {
    client_id: '9b8e604efcd26ebbdedd',
    client_secret: 'd641e81ad3029812cf4c78360a2dee32414659a4', 
  },
})
gitment.render('container')
</script>