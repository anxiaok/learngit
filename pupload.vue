<!--
    使用组件说明：
        1，local_name:文件原始名字
        2，random_name:文件随机名字

    如何调用：
        <plup-load :browse_button="'addAppendix'" @getImgList_addAppendix = "getImgList_addAppendix" :type="'getImgList_addAppendix'"></plup-load>

        1,browse_button:触发按钮
        2，@getImgList_addAppendix = "getImgList_addAppendix"：触发事件
        3，:type="'getImgList_addAppendix'"：触发类型，
    注意：
        1，browse_button一定是能找到dom

-->
<template>
    <div style="display: none">
        <form name=theform>
            <!--可以设置不同的文件名上传-->
            <input type="radio" name="myradio" value="local_name" checked=true/> 上传文件名字保持本地文件名字
            <input type="radio" name="myradio" value="random_name" /> 上传文件名字是随机文件名
            <br/>
            上传到指定目录:<input type="text" id='dirname' placeholder="如果不填，默认是上传到根目录" size=50>
        </form>
    </div>
</template>
<script>
    import plupload from 'plupload'
    export default{
        props:{
            browse_button: '',
            type:'',
            dropBtn:''
        },
        data(){
            return {
                token:this.$cookie.get('token'),
                parames:'', // 后台返回的直传签名
                accessid: '', // 全局OSSAccessKeyId
                host : '', // 后台返回的host
                policyBase64 : '', // policey
                signature : '', // 签名
                callbackbody:'', // 回调
                g_dirname:'', // 全局根目录（未实现）
                g_object_name:'', // 全局key名字
                g_object_name_type:'', // 上传文件名是随机还是原名字
                uploader:'', // uploader实例
                fileLen:0, // 文件个数
            }
        },
        components:{
            plupload
        },
        mounted(){
            this.initPlupLoad();
        },
        methods: {
            // 从后台获取asign
            getSign(){
                this.$request.apiAxiosHasBaseUrl('get','cmsApp/content/getSign',{},{
                    token:this.token
                },false).then((request) => {
                    let requestData = request.data;
                    this.accessid = requestData.accessid;
                    this.signature = requestData.signature;
                    this.policyBase64 = requestData.policy;
                    this.host = requestData.host;
                    this.g_object_name = requestData.dir;
                    this.callbackbody = requestData.callback;
                    this.parames = {
                        'key' : requestData.dir,
                        'policy':requestData.policy,
                        'OSSAccessKeyId':requestData.accessid,
                        'success_action_status' : '200', //让服务端返回200,不然，默认会返回204
                        'callback': requestData.callback,
                        'signature': requestData.signature,
                        'host':requestData.host
                    };
                    this.initPlupLoad();
                })
            },
            check_object_radio() {
                let tt = document.getElementsByName('myradio');
                for (let i = 0; i < tt.length ; i++ )
                {
                    if(tt[i].checked)
                    {
                        this.g_object_name_type = tt[i].value;
                        break;
                    }
                }
            },
            get_dirname(){
                let dir = document.getElementById("dirname").value;
                if (dir != '' && dir.indexOf('/') != dir.length - 1){
                    dir = dir + '/'
                }
                this.g_dirname = dir
            },
            random_string(len) {
                len = len || 32;
                const chars = 'ABCDEFGHJKMNPQRSTWXYZabcdefhijkmnprstwxyz2345678';
                let maxPos = chars.length;
                let pwd = '';
                for (let i = 0; i < len; i++) {
                    pwd += chars.charAt(Math.floor(Math.random() * maxPos));
                }
                return pwd;
            },
            get_suffix(filename) {
                let pos = filename.lastIndexOf('.')
                this.suffix = ''
                if (pos != -1) {
                    this.suffix = filename.substring(pos)
                }
                return this.suffix;
            },
            calculate_object_name(filename){
                this.g_object_name = this.parames.key;
                if (this.g_object_name_type == 'local_name')
                {
                    this.g_object_name += "${filename}"
                }
                else if (this.g_object_name_type == 'random_name')
                {
                    this.suffix = this.get_suffix(filename)
                    this.g_object_name = this.g_object_name + this.random_string(10) + this.suffix
                }
                return ''
            },
            get_uploaded_object_name(filename){
                if (this.g_object_name_type == 'local_name')
                {
                    let tmp_name = this.g_object_name
                    tmp_name = tmp_name.replace("${filename}", filename);
                    return tmp_name
                }
                else if(this.g_object_name_type == 'random_name')
                {
                    return this.g_object_name
                }
            },
            set_upload_param(up, filename, ret){
                if (filename != '') {
                    this.suffix = this.get_suffix(filename)
                    this.calculate_object_name(filename)
                }
                let new_multipart_params = {
                    'key' : this.g_object_name,
                    'policy': this.policyBase64,
                    'OSSAccessKeyId': this.accessid,
                    'success_action_status' : '200', //让服务端返回200,不然，默认会返回204
                    'callback' : this.callbackbody,
                    'signature': this.signature,
                };
                if(this.type === 'getImgList_addBanner'){
                    up.setOption({
                        // 'drop_element':this.dropBtn,
                        'url': this.host,
                        'multipart_params': new_multipart_params,
                    });
                } else {
                    up.setOption({
                        'url': this.host,
                        'multipart_params': new_multipart_params
                    });
                }

                up.start();
            },
            initPlupLoad(){
                const that = this;
                this.uploader = new plupload.Uploader({
                    runtimes : 'html5,flash,silverlight,html4',
                    browse_button : this.browse_button,
                    url : 'http://oss.aliyuncs.com',
                    max_file_size : '100mb', //最大只能上传10mb的文件
                    init: {
                        FilesAdded: function(up, files) {
                            that.$request.apiAxiosHasBaseUrl('get','cmsApp/content/getSign',{},{
                                token:that.token
                            },false).then((request) => {
                                let requestData = request.data;
                                that.accessid = requestData.accessid;
                                that.signature = requestData.signature;
                                that.policyBase64 = requestData.policy;
                                that.host = requestData.host;
                                that.g_object_name = requestData.dir;
                                that.callbackbody = requestData.callback;
                                that.parames = {
                                    'key' : requestData.dir,
                                    'policy':requestData.policy,
                                    'OSSAccessKeyId':requestData.accessid,
                                    'success_action_status' : '200', //让服务端返回200,不然，默认会返回204
                                    'callback': requestData.callback,
                                    'signature': requestData.signature,
                                    'host':requestData.host
                                };
                                up.start();
                            })
                        },
                        BeforeUpload: function(up, file) {
                            that.check_object_radio();
                            that.get_dirname();
                            that.set_upload_param(up, file.name, true);
                        },
                        UploadProgress: function(up, file) {},
                        FileUploaded: function(up, file, info) {
                            if (info.status == 200){
                                let fileName = that.get_uploaded_object_name(file.name);
                                that.$emit(that.type,{imgUrl:`${that.parames.host}/${fileName}`,fileLen:that.fileLen,fileName:fileName,name:file.name,size:file.size});
                            }else if (info.status == 203){
                                console.log(info.response)
                            }
                        },
                        Error: function(up, err) {
                            if (err.code == -600) {
                                console.log("选择的文件太大了或文件为空,可以根据应用情况，在upload.js 调整设置");
                            }
                            else if (err.code == -601) {
                                console.log("选择的文件后缀不对,可以根据应用情况，在upload.js进行设置可允许的上传文件类型");
                            }
                            else if (err.code == -602) {
                                console.log("这个文件已经上传过一遍了");
                            }else{
                                console.log(err.response);
                            }
                        }
                    }
                });
                this.uploader.init();
                if(this.type === 'getImgList_addBanner') {
                    this.$emit('getUploader', this.uploader)
                }
            }
        }
    }
</script>
