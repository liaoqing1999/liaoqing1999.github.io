---
layout: post
title: element文件上传
date: 2020-07-17 17:11:17
tags: vue
categories: vue
---

# element文件上传

## 前言

今天进行了element 文件上传组件的运用，写一下心得

## 前端组件配置

具体组件配置官方文档里面有，我就不一一说明了

```
//自定义上传
<el-upload id="el-upload" 
    ref="uploadForm" 
    :auto-upload="false" 
    :http-request="upLoad" 
    :on-remove="onRemove" 
    :before-upload="beforeUpload"
    > 
     <el-button slot="trigger" size="small" type="primary">
      点击上传
     </el-button> 
    </template> 
   </el-upload> 
```

这里主要说明两种情况：

第一种直接用action配置上传地址，这种的优点就是方便、省心，element会直接把文件调用后端接口，但是不够灵活，无法传递额外的参数，也只能一个个文件上传。

第二种用http-request覆盖掉默认的上传方式，可以在这里调用后端地址，配置相关请求头，增加参数等等，这里可以设置取消自动上传，然后通过按钮触发组件的上传方法，但是这种方法其实也是一个个发请求的。

而且我的需求是需要在点击提交的时候根据输入的编码传给后端，然后生成文件夹的名字，所有还需要进行一些改造。

```
//我的上传方法并没有写任何东西，因为我的上传是通过点击提交的时候再自定义上传的
//之所以要写一个空方法是因为需要获取到组件的file数组
//如果不写这个方法  会取action里面的值来进行上传（就算action为空也会）
upLoad(params){
	
},
```

```
//fileList是我定义的用于存储file数组的变量
//在上传前，将文件存储到fileList里面
beforeUpload(file){
	this.fileList.push(file)
},
//移除方法，将fileList里面的file去掉
onRemove(file){
	let index = this.fileList.findIndex(fileItem => fileItem.uid === file.uid);
	this.fileList.splice(index, 1);
}
```

```
//点击提交的时候触发的方法
//第一句话是触发上传组件的上传方法 这样才能触发beforeUpload（因为我们写了取消自动上传）
this.$refs.uploadForm.submit()
//对fileList用formData拼接
let formData = new FormData();
this.fileList.forEach((item)=>{
    formData.append("file",item);
})
//这里设置下请求头和传递的额外参数
$http.post("bmp/v1/chaincode/uploadEnclosure/"+this.data.code, formData,{headers: {'Content-Type': 'multipart/form-data'}}).then(data => {
	console.log(data)
}).catch(error => {
    	this.$message({
        message: "error",
        type: 'error'
        })
    })
```

现在前端基本配置好了，调用接口就可以实现文件上传了

## 后端接口

controller

```
	@POST
	@Path("/chaincode/uploadEnclosure/{code}")
	@Consumes(MediaType.MULTIPART_FORM_DATA)
	@ApiOperation("文件上传")
	public List<Map<String, Object>> uploadEnclosure(@Context HttpServletRequest request,@PathParam(value = "code") String code) {
		StandardMultipartHttpServletRequest standardRequest = new StandardMultipartHttpServletRequest(request);
		MultiValueMap<String, MultipartFile> multiFileMap = standardRequest.getMultiFileMap();
		List<MultipartFile> fileList = null;
		for (String key : multiFileMap.keySet()) {
			fileList = multiFileMap.get(key);
		}
		if(null == fileList || fileList.size()==0) {
			throw new Exception("文件为空");
		}
		List<Map<String, Object>> result = new ArrayList<Map<String, Object>>();
		String realPath = request.getServletContext().getRealPath("/");
		String baseUrl = realPath+code + File.separator;
		for(MultipartFile file : fileList) {
			Map<String, Object> uploadFile = fileService.uploadFile(file,baseUrl,  file.getOriginalFilename());
			result.add(uploadFile);
		}
	
		return result;	
	}
```

```
//另外种方式
 /**
     * 实现多文件上传
     **/
    @RequestMapping(value = "/multifile", method = RequestMethod.POST)
    @ResponseBody
    public ResultUtil multifileUpload(
            @RequestParam("fileName") List<MultipartFile> files) {
        if (files.isEmpty()) {
            return ResultUtil.failed("文件为空");
        }
        for (MultipartFile file : files) {
            String fileName = file.getOriginalFilename();
            int size = (int) file.getSize();
            logger.info(fileName + "-->" + size);

            if (file.isEmpty()) {
                return ResultUtil.failed("文件为空");
            } else {
                File dest = new File(path + "/" + fileName);
                if (!dest.getParentFile().exists()) {
                    dest.getParentFile().mkdir();
                }
                try {
                    file.transferTo(dest);
                } catch (Exception e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                    return ResultUtil.failed();
                }
            }
        }
        return ResultUtil.success(null);
    }

```

service

```
public Map<String, Object> uploadFile(MultipartFile file,String baseUrl,String fileName) {
		if (file.isEmpty()) {
			throw new Exception("文件为空");
        }
        File dest = new File(baseUrl+fileName);
		if (!dest.getParentFile().exists()) {
			dest.getParentFile().mkdirs();
        }
        try {
            file.transferTo(dest);
            Map<String,Object> result =new HashMap<String,Object>();
            result.put("name", fileName);
            String hash = getFileSHA1(dest);
            result.put("hash", hash);
            return result;
        } catch (IllegalStateException e) {
            e.printStackTrace();
            throw new Exception("文件为空");
        } catch (IOException e) {
            e.printStackTrace();
            throw new Exception("文件为空");
        } 
	}
```

## sha256获取文件hash

这些方法也是写在service里面的

```
private static String getFileSHA1(File file) {
        String str = "";
        try {
            str = getHash(file, "SHA-256");
        } catch (Exception e) {
            e.printStackTrace();
        }
        return str;
    }
	
	private static String getHash(File file, String hashType) throws Exception {
        InputStream fis = new FileInputStream(file);
        byte buffer[] = new byte[1024];
        MessageDigest md5 = MessageDigest.getInstance(hashType);
        for (int numRead = 0; (numRead = fis.read(buffer)) > 0; ) {
            md5.update(buffer, 0, numRead);
        }
        fis.close();
        return toHexString(md5.digest());
    }
	
	 private static String toHexString(byte b[]) {
        StringBuilder sb = new StringBuilder();
        for (byte aB : b) {
            sb.append(Integer.toHexString(aB & 0xFF));
        }
        return sb.toString();
	 }
```

