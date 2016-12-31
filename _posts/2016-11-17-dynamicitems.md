---
layout:     post
title:      "Dynamic Tables And Rows"
subtitle:   "Add elements dynamicly"
date:       2016-11-16 19:22:00+0800
author:     "ZanXus"
header-img: "img/blog/header/post-bg-02.jpg"
thumbnail: /img/blog/thumbs/thumb02.png
tags: [jquery]
category: [javascript]
comments: true
share: true
---


#  The page Template engine is velocity.

## Below is the code segment in vm.

~~~ html
<div class="control-group">
    <label class="control-label">Sku规格参数</label>
    <div class="controls" id="spec_params">
        <div>
            <button type="button" class="btn btn-primary" onclick="addSpec()">添加规格</button>
        </div>
        <hr>
    #* If the operation is edit not add,then echo parameters into page*#
        #if(${param.skuParamsContent})
            <script type="text/javascript">
                 param =${param.skuParamsContent};
            </script>
        #end
    </div>
</div>
~~~

## Here's the jQuery code.

~~~ javascript
$(document).ready(function () {
    if (typeof (param)=="object"&&Object.prototype.toString().toLowerCase()=="[object object]"&&!param.length){
        echoParams(param);
    }
});

/**
 * echo params into page
 *
 */
function  echoParams(paramsContent) {
    var specLen=$(".specs").length;
    $.each(paramsContent,function (key,value) {
        var specId="spec"+specLen;
        $("#spec_params").append("<div id="+specId+" class='specs'>" +
            "<div><input type='text' value='"+key+"' style='width: 29%'/><span style='padding-left: 25px;'>" +
            "<button id='btn"+specLen+"' type='button' class='btn btn-success'  onclick='addParam(this.id)'>添加参数</button>" +
            "&nbsp;&nbsp;&nbsp;" +
            "<button type='button' id="+specId+"_btn class='btn btn-warning' onclick='delSpec(\""+(specId)+"\","+(specLen)+")'>删除规格</button>" +
            "</span> " +
            "</div>" +
            "<hr>" +
            "<table id='table_btn"+specLen+"'>" +
            "<tbody>" +
            "</tbody>" +
            "</table><hr>" +
            "</div>");
        var params=paramsContent[key];
        for (var param in params){
            $.each(params[param],function (k,v) {
                var tableName = "table_btn" +specLen ;
                var trLen = $("#" + tableName + " tr").length;
                $("#" + tableName).append("<tr id=" + tableName + "_" + trLen + " class='" + tableName + "'>"
                    + "<td><input name='key' type='text' value='"+k+"'/></td>" +
                    "<td><input name='value' type='text' value='"+v+"'/></td>" +
                    "<td><button  name='"+tableName+"_"+trLen+"' type='button' class='btn btn-danger' onclick='delParam(this.name,"+(trLen)+")'>删除参数</button></td>" +
                    "</tr>");
            });
        }
        specLen++;
    });
}


/**
 * add spec dynamicly
 *
 */
function addSpec() {
    var specLen=$(".specs").length;
    var specId="spec"+specLen;
    $("#spec_params").append("<div id="+specId+" class='specs'>" +
        "<div><input type='text' value='' style='width: 29%'/><span style='padding-left: 25px;'>" +
        "<button id='btn"+specLen+"' type='button' class='btn btn-success'  onclick='addParam(this.id)'>添加参数</button>" +
        "&nbsp;&nbsp;&nbsp;" +
        "<button type='button' id="+specId+"_btn class='btn btn-warning' onclick='delSpec(\""+(specId)+"\","+(specLen)+")'>删除规格</button>" +
        "</span> " +
        "</div>" +
        "<hr>" +
        "<table id='table_btn"+specLen+"'>" +
        "<tbody>" +
        "</tbody>" +
        "</table><hr>" +
        "</div>"
    );
}

/**
 * add rows in table dynamicly
 * @param btnId
 */
function addParam(btnId) {
    var tableName = "table_" + btnId;
    var trLen = $("#" + tableName + " tr").length;
    $("#" + tableName).append("<tr id=" + tableName + "_" + trLen + " class='" + tableName + "'>"
        + "<td><input name='key' type='text' value=''/></td>" +
        "<td><input name='value' type='text' value=''/></td>" +
        "<td><button  name='"+tableName+"_"+trLen+"' type='button' class='btn btn-danger' onclick='delParam(this.name,"+(trLen)+")'>删除参数</button></td>" +
        "</tr>");
}


/**
 * delete rows in table dynamicly
 * @param btnName
 * @param index
 */
function delParam(btnName,index) {
    var tr=$("tr[id="+btnName+"]");
    // var trId=tr.attr('id');
    var trClass=tr.attr('class');
    var trLen = $("." + trClass).length;
    for (var i = index , j = trLen; i < j; i++) {
        var nextKeyVal = $("#" + trClass+"_"+(i+1) + " input[name='key']").val();
        var nextValueVal = $("#" + trClass+"_"+(i+1) + " input[name='value']").val();
        $("tr[id=" + trClass+"_"+i+ "]").replaceWith("<tr id=" + trClass + "_" +i + " class='" + trClass + "'>"
            + "<td><input name='key' type='text' value='" + nextKeyVal + "'/></td>" +
            "<td><input name='value' type='text' value='" + nextValueVal + "'/></td>" +
            "<td><button name='"+trClass+"_"+i+"' type='button' class='btn btn-danger' onclick='delParam(this.name,"+i+")'>删除参数</button></td>" +
            "</tr>");
    }
    $("tr[id=" + trClass +"_"+(trLen-1)+ "]").remove();
}


/**
 * delete spec and change the attribute values of other specs after deleting(all id or class index subtract 1)
 * @param specId
 * @param index
 */
function delSpec(specId,index) {
    var result=confirm("确定要删除该规格吗？");
    if(result){
        $("#"+specId).remove();
        var specLen=$(".specs").length;
        for(var n=index,j=specLen;n<j;n++){
            var i=n+1;
            console.log(specId);
            console.log("tr length=="+$("#table_btn"+i+" tr").length);
            $("#table_btn"+i+" tr").each(function (index1) {
                var trId="table_btn"+i+"_"+index1;
                $("button[name="+trId+"]").attr("name","table_btn"+(i-1)+"_"+index1);
                $("#"+trId).attr("class","table_btn"+(i-1));
                $("#"+trId).attr("id","table_btn"+(i-1)+"_"+index1);
            });
            $("#table_btn"+i).attr("id","table_btn"+(i-1));
            $("#btn"+i).attr("id","btn"+(i-1));
            $("#spec"+i+"_btn").attr("onclick","delSpec('spec"+(i-1)+"',"+(i-1)+")");
            $("#spec"+i+"_btn").attr("id","spec"+(i-1)+"_btn");
            $("#spec"+i).attr("id","spec"+(i-1));
        }
    }
}

/**
* generate all created elements data into json for posting data to back-end with ajax
*/
function  makeJson() {
     var array={};
     var reslut={};
    $("#spec_params  .specs").each(function (index1) {
        console.log("specs length=="+$("#spec_params  .specs").length);
        var specs={};
        var items=[];
        var specName="";
        $(".specs:eq("+index1+") input[type='text']").each(function (index) {
            var len=$(".specs:eq("+index1+") input[type='text']").length;
            console.log(len);
            if("undefined"==typeof ($(this).attr("name"))){
                specName=$(this).val();
            }
            else if("key"==$(this).attr("name")){
                var item={};
                item[$(this).val()]=$(".specs:eq("+index1+") input[type='text']").eq(index+1).val();
                items.push(item);
            }
        });
        reslut[specName]=items;
    });
    array["data"]=reslut;
    array["paramId"]=$("#param_id").html();
    return JSON.stringify(array);
}
~~~




