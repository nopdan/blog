---
title: '更多'
date: 2021-08-16T18:55:07+08:00
toc: false
---

## 关于我

> 爱一定存在与世上，一定存在  
> 无从寻觅的是爱的表现，是它的表达方式

- **关键词**：00 后，重庆大学数学系，编程，输入法，游戏，音乐，追番，up 主。
- **输入法**：星辰双拼，星辰星笔，小可两笔，倾心两笔。
- **音乐**：流行，民谣，古风，轻音乐，古典，电音，日语，acg。
- **游戏**：只玩 CF ，主玩跳跳乐模式，有在 b 站发视频。
- **语言**：普通话，重庆话，英语 4 级。
- **编程**：python，golang。

## 友链

> 感谢 @芝士部落格 提供了友链页面模板~

在友链形成的网络中漫游，是一件很有意思的事情。

**以前的人们通过信笺交流，而我们通过友链串联起一个「世界」。希望你我都能在这个「世界」中有所收获**

**注：** <span style="color:red;">下方友链次序每次刷新页面随机排列。<span>

<div class="linkpage"><ul id="friendsList"></ul></div>

## 交换友链

如果你觉得我的博客有些意思，而且也有自己的独立博客，欢迎与我交换友链~

可在评论区提交友链申请，格式如下：

    站点名称：nopdan's blog
    作者: @单单
    站点地址：https://nopdan.com/
    个人形象：https://nopdan.com/avatar.webp
    站点描述：但知行好事，莫要问前程

<script type="text/javascript">
// 以下为样例内容，按照格式可以随意修改
var myFriends = [
    ["https://blog.weearc.top/", "https://blog.weearc.top/image/face.ico", "@weearc", "相离相惜莫相忘，且行且歌且珍惜"],
    ["https://www.yunyitang.me/zh/", "https://www.yunyitang.me/img/Avatar.png", "Yunyi’s Blog", "Little squirrel Hopping around"],
    ["https://rea.ink/", "https://rea.ink/logo.png", "@倾书", "清风皓月，光景常新"],
    ["https://thiscute.world/", "https://thiscute.world/avatar/myself.jpg", "@Ryan4Yin", "我错过花，却看见海。"],
    
];


// 以下为核心功能内容，修改前请确保理解您的行为内容与可能造成的结果
var  targetList = document.getElementById("friendsList");
while (myFriends.length > 0) {
    var rndNum = Math.floor(Math.random()*myFriends.length);
    var friendNode = document.createElement("li");
    var friend_link = document.createElement("a"), 
        friend_img = document.createElement("img"), 
        friend_name = document.createElement("h4"), 
        friend_about = document.createElement("p")
    ;
    friend_link.target = "_blank";
    friend_link.href = myFriends[rndNum][0];
    friend_img.src=myFriends[rndNum][1];
    friend_name.innerText = myFriends[rndNum][2];
    friend_about.innerText = myFriends[rndNum][3];
    friend_link.appendChild(friend_img);
    friend_link.appendChild(friend_name);
    friend_link.appendChild(friend_about);
    friendNode.appendChild(friend_link);
    targetList.appendChild(friendNode);
    myFriends.splice(rndNum, 1);
}
</script>

<style>

.linkpage ul {
    color: rgba(255,255,255,.15)
}

.linkpage ul:after {
    content: " ";
    clear: both;
    display: block
}

.linkpage li {
    float: left;
    width: 48%;
    position: relative;
    -webkit-transition: .3s ease-out;
    transition: .3s ease-out;
    border-radius: 5px;
    line-height: 1.3;
    height: 90px;
    display: block
}

.linkpage h3 {
    margin: 15px -25px;
    padding: 0 25px;
    border-left: 5px solid #51aded;
    background-color: #f7f7f7;
    font-size: 25px;
    line-height: 40px
}

.linkpage li:hover {
    background: rgba(230,244,250,.5);
    cursor: pointer
}

.linkpage li a {
    padding: 0 10px 0 90px
}

.linkpage li a img {
    width: 60px;
    height: 60px;
    border-radius: 50%;
    position: absolute;
    top: 15px;
    left: 15px;
    cursor: pointer;
    margin: auto;
    border: none
}

.linkpage li a h4 {
    color: #333;
    font-size: 18px;
    margin: 0 0 7px;
    padding-left: 90px
}

.linkpage li a h4:hover {
    color: #51aded
}

.linkpage li a h4, .linkpage li a p {
    cursor: pointer;
    white-space: nowrap;
    text-overflow: ellipsis;
    overflow: hidden;
    line-height: 1.4;
    margin: 0 !important;
}

.linkpage li a p {
    font-size: 12px;
    color: #999;
    padding-left: 90px
}

@media(max-width: 460px) {
    .linkpage li {
        width:97%
    }

    .linkpage ul {
        padding-left: 5px
    }
}

</style>
