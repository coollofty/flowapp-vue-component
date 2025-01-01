<template>
    <div class="blocks-generator">
        <!--当没有任何工作流被绑定的时候，使用default插槽-->
        <slot v-if="!lastUIData"></slot>

        <!--加载中的状态，如获取工作流信息-->
        <slot v-if="loading && !generateData" name="loading"></slot>

        <!--上传文件中的状态，-->
        <slot v-if="uploadProgress !== null && !generateData" name="uploading" v-bind:data="uploadProgress"></slot>
        
        <!--所有inputs数据的渲染-->
        <slot v-if="lastUIData && !generateData" name="inputs" v-bind:data="lastUIData"></slot>

        <!--生成状态-->
        <slot v-if="generateData" name="generating" v-bind:data="generateData"></slot>

        <!--最后一次的生成结果展示-->
        <slot v-if="lastGenResult" name="result" v-bind:data="lastGenResult"></slot>
    </div>
</template>

<style taskDatad>
.blocks-generator { width:100%; }
</style>

<script>
/* 积木AI的Vue生成组件
 * 所有的API请求返回结构都有一个基础的标准结构，如下 ：
 * {
 *   success: true | false,     // 是否成功。这里的成功指的是接口运行时检验和逻辑代码没有报错，和网络的状态没有关系
 *   code: 400,                 // 如果失败，就会有这个失败代码，是一个数字。如果成功的话code不存在
 *   message: '失败的简单描述',  // 仅在失败的时候才会存在，提示了大概的原因
 *   data: { }                  // 接口返回的结果，data里的数据因具体接口而异，如果失败的话这个data成员是不存在的
 */

// 语言包（建议）。这是所有可能出现的状态对应的中英文翻译的建议，请根据自己的需要进行更改
const _Languages = {
    'zh-cn': {
        create: '任务已创建等待运行',
        queue: '任务排队中',
        wait: '数据预处理或文件下载中',
        started: '生成已开始',
        updating: '生成进行中',
        uploading: '正在回传结果',
        downloading: '文件下载中',
        error: '生成出错',
        timeout: '生成超时',
        aborted: '生成被用户终止',
        aborting: '生存正在尝试终止'
    },

    'en': {
        create: 'Task created',
        queue: 'Task queued, please wait',
        wait: 'Preprocessing or Downloading data',
        started: 'Generation started',
        updating: 'Generation running',
        uploading: 'Generation has ended, waiting for result(s)',
        downloading: 'File (result) downloading',
        error: 'Generation failed',
        timeout: 'Generation timeouted',
        aborted: 'Generation terminated by user',
        aborting: 'Generation is attempting to terminate'
    }
}

export default {
    name: 'BlocksGenerator',
    props: {
        // API基础URL，一般使用默认值
        apiBaseUrl: {
            type: String,
            default: 'https://api.aibot.wang'
        },
        // 文件服务器的基础URL，如果未指定的话则同apiBaseUrl
        fileBaseUrl: String,

        // token和apiKey必须选一个，否则无法调用start函数
        token: String,
        apiKey: String,

        // 如果为true，则最后一次调用bindWorkflow后会激活inputs插槽，并将inputs数据结构传入，需要该插件显示所有需要用户填充的输入数据
        renderBindUI: Boolean,
    },
    emits: ['ended', 'statusChanged'],
    watch: {
        token(val) {
            if (val) this._updateHeaders();
        },
        apiKey(val) {
            if (val) this._updateHeaders();
        }
    },
    data() {
        return {
            loading: false,
            uploadProgress: null,
            generateData: null,
            lastGenResult: null,
            lastUIData: null,    
            defLangPack: _Languages        
        }
    },
    mounted() {
        this.cachedWorkflows = {}

        this._updateHeaders();
    },
    methods: {
        /** 上传一个文件为资源文件，文件可以是图片、视频、音频
         * @param {File} file 文件对象，可以是<input type="file" />的value也可以是通过其它方式得来的单个文件对象（如拖放操作、window.showOpenFilePicker函数等）
         * @param {String} type 资源类型，只能是image、video、audio这3种中的一种，其它值报错
         * @param {Object} options 附加选项，完整定义为 { progressCallback: function(percent, completed, total) {} }
         * @param (boolean) returnFullObj 是否返回完整的资源信息，如果为true，则返回完整的资源信息结构（所含字段较多），否则只返回id、type、fileSize、url这4个字段         
         * @return {Object} 如果函数调用失败（配置或参数不正确）返回null，否则成功返回上传文件后所得的资源信息，该结构体可能较大，但主要有用的一般有{ id、type, fileSize、url }，失败则返回标准失败格式{ success: false, code: xxx, message: '错误信息' }
         */
        uploadFile(file, type, options, returnFullObj) {
            const typeToResType = { image: 'picture', video: 'video', audio: 'audio' }, formData = new FormData(), opts = options || {};

            if (!this.headers) {
                console.error('Please set token or apiKey first');
                return;
            }
            if (!(file instanceof File)) {
                console.error('uploadFile file must be a File instance');
                return;
            }
            if (!typeToResType[type]) {
                console.error('uploadFile type must be image, video or audio');
                return;
            }

            formData.set('file', file);
            formData.set('type', typeToResType[type]);          // 必须正确设置文件的类型
            formData.set('md5', '1');                           // 启动文件Hash检测，当文件已经存在（Hash相同）时，不会重复保存该文件
			formData.set('autoFilename', '1');                  // 上传后的文件名就自动生成吧
			formData.set('category', 'temp');                   // 生成前用的文件，将其归为临时类目即可
			formData.set('accessType', 'public');               // 必须是公开的，否则不同的生成器有可能在获取文件的时候失败
            formData.set('storageDays', '1');                   // 用于生成任务的文件一般都是临时文件，1天的有效期足够了

            if (file.name) {
                const pos = file.name.lastIndexOf('.');
                if (pos != -1) {
                    formData.set('fileExtension', file.name.slice(pos + 1));
                }

                formData.set('title', file.name);               // 仅仅是记录一下上传时所选文件的原始文件名
            }

            if (opts.values instanceof Object) {
                for(const k in opts.values) {
                    formData.set(k, opts.values[k]);
                }
            }

            const baseUrl = this.fileBaseUrl || this.apiBaseUrl;

            this.uploadProgress = null;
            return new Promise(resolve => {
                var xhr = new XMLHttpRequest();

                xhr.upload.addEventListener('progress', function(e){
                    if (e.lengthComputable) {
                        const val = e.loaded / e.total * 100.0;
                        if (opts.progressCallback) {
                            opts.progressCallback(val, e.loaded, e.total);
                        } else {
                            this.uploadProgress = { progress: val, completion: e.loaded, total: e.total };
                        }
                    }
                }, false);

                xhr.open('POST', `${baseUrl}/resources`);
                if (this.token) xhr.setRequestHeader('Authorization', this.token);
                else if (this.apiKey) xhr.setRequestHeader('WXCZ-ACCESS-KEY', this.apiKey);
                xhr.send(formData);

                xhr.onreadystatechange = () => {
                    if (xhr.status === 200) {
                        if (xhr.readyState === 4) {
                            this.uploadProgress = null;

                            try {
                                let rObj = JSON.parse(xhr.responseText);
                                if (rObj.success) {
                                    const simpleObj = {}
                                    for(const nm of ['id', 'type', 'fileSize', 'url']) {
                                        simpleObj[nm] = rObj.data[nm];
                                    }

                                    rObj.data = simpleObj;                                    
                                }
                                
                                resolve(rObj);
                            } catch(e) {
                                resolve({ success: false, code: 400, message: e.toString() });
                            }
                        }

                        return;
                    }

                    this.uploadProgress = null;
                    resolve({ success: false, message: 'upload file failed' });
                }
            });
        },

        /** 删除已成功上传为资源的文件
         * @param {Object} res 资源对象，也可以是资源的ID
         */
        async deleteResource(res) {
            const resId = typeof res == 'number' ? res : res.id;
            try {
                let r = await fetch(`${this.apiBaseUrl}/resources/${resId}`, {
                    method: 'DELETE',
                    cache: 'no-cache',
                    headers: this.headers
                });

                return await r.json();
            } catch(e) {
                console.error(e);
                return { success: false, code: 400, message: e.toString() };
            }
        },

        /** 绑定工作流的UI数据
         * @param {String|Number} workflowId 工作流的ID，请先在flowapps.store上注册一个工作流，并发布完成，
         * @return {Array} 返回为所有输入数据描述数组，基本格式示例：[
         *  { type: 'image', id: '节点ID', value: '初始值' }
         * ]
         */
        async bindWorkflow(workflowId) {
            let err = { success: false, code: 400, message: '' }
            if (this.uploadProgress !== null) {
                console.warn(err.message = 'Please wait for the previous file uploading finished');
                return err;
            }

            if (typeof workflowId == 'string' && !/^[\d]+$/.test(workflowId)) {
                console.error(err.message = 'workflowId must be a number or a numerical string composed of 0-9 character(s)');
                return err;
            }

            if (!this.headers) {
                console.error(err.message = 'Please set token or apiKey first');
                return err;
            }

            const caches = this.cachedWorkflows;
            let uiData = caches[workflowId];

            if (!uiData) {
                this.loading = true;

                try {
                    let r = await fetch(`${this.apiBaseUrl}/workflows/${workflowId}?fields=uiData`, {
                        method: 'GET',
                        cache: 'no-cache',
                        headers: this.headers
                    });

                    r = await r.json();
                    if (!r.success) {
                        this.loading = false;
                        return r;
                    }

                    // 将输入的绑定数据解析出来
                    uiData = r.data.uiData;
                    if (typeof uiData == 'string') {
                        uiData = JSON.parse(uiData);
                    }

                    caches[workflowId] = uiData = uiData.uiData;
                    
                } catch(e) {
                    console.error(e);
                    this.loading = false;
                    return { success: false, code: 400, message: e.toString() };
                }

                this.loading = false;
            }

            if (this.renderBindUI)
                this.lastUIData = [ ...uiData ];

            return { success: true, data: uiData }
        },

        /** 开始一个生成式任务
         * @param {String|Number} workflowId 工作流的ID，请先在flowapps.store上注册一个工作流，并发布完成，
         * @param {Array} inputs 输入的数据，格式为 [ { id: '对应bindWorkflow函数返回值中某一项的节点ID', value: '根据节点type决定的值，如果是文件类型，则必须是资源对象，如果是字符串、数值、布尔型，则必须是对应的值' } ]
         */
        async start(workflowId, inputs) {
            const uiData = await this.bindWorkflow(workflowId); 
            if (!uiData)
                return;

            const posts = {
				plat: 'comfyui',
				private: true,
				stepEvent: true,
				standaloneMode: true,
				channelName: 'magicps',
				workflow: parseInt(workflowId),
				params: {
					isFullJson: true,
					workflowName: 'workflow-#' + workflowId,
					resultUpload: {
                        storageDays: 7,                                     // 生成的结果文件会存储7天，之后自动删除
                        category: 'generated'
                    },
					imageCreation: {
                        byModel: 'workflow-#' + workflowId
                    },
					valuesMap: {},
				}
			}

            let params = posts.params, res, input, err = { success: false, code: 400, message: '' };
			
			for(const decl of uiData.data) {
                res = {}

                if (Array.isArray(inputs)) {
                    input = inputs.find(chk => chk.id === decl.id);
                    if (!input) {
                        console.error(err.message = `Input ${decl.id} have no input data, please setup first`);
                        return err;
                    }
                } else {
                    input = decl;
                }

				switch(input.type) {
					case 'image':
					case 'video':
					case 'audio':
                        res = {}
                        if (input.value instanceof Object) {                            
                            res.url = input.value.url;
                            res.fileSize = input.value.fileSize;                      
                        }

                        if (!res.url) {
                            console.error(err.message = `Input ${input.id} is a input file type, but value is not a resource object`);
                            return err;
                        }
						break;
						
					case 'text':
					case 'string':
						if (typeof input.value != 'string') {
							console.error(err.message = `Input ${input.id} is a string type, but value is not a string`);
                            return err;
						}
						break;

                    case 'boolean':
                        if (typeof input.value != 'boolean') {
							console.error(err.message = `Input ${input.id} is a string type, but value is not a boolean`);
                            return err;
						}
                        break;

                    case 'number':
                        if (typeof input.value != 'boolean' || isNaN(input.value)) {
							console.error(err.message = `Input ${input.id} is a string type, but value is not a boolean`);
                            return err;
						}
                        break;
				}

				for(const key of ['id', 'up', 'type']) {
					res[key] = input[key];
				}
				
				params.valuesMap[input.id] = res;
			}

            // 提交任务
            this.generateData = {
				status: 'create', lastProgress: null
			}

            try {
                let r = await fetch(`${this.apiBaseUrl}/workProgresses`, {
                    method: 'POST',
                    headers: this.headers,
                    body: JSON.stringify(posts)
                });

                r = await r.json();

                if (!r.success) {
                    this.generateData = null;
                    return r;
                }

                // 使用localStorage存储任务ID，以便中断后再续状态
                this.executingTasks = localStorage.getItem('generateData_task');

                if (this.executingTasks) this.executingTasks = JSON.parse(this.executingTasks);
                else this.executingTasks = {};

                const taskData = { wkId: r.data.id, ts: Date.now() }
                this.executingTasks[workflowId] = taskData;

                localStorage.setItem('generateData_task', JSON.stringify(this.executingTasks));

                // 任务一旦提交成功，就肯定会进入队列状态
                this.generateData.status = 'queue';
                this.generateData.wkId = r.data.id;
                // 1秒后进入Event检查
			    taskData.checkTM = setTimeout(() => this._checkProgress(workflowId, r.data.id), 1000);

                return r;

            } catch(e) {
                console.error(e);
                this.generateData = null;
                return { success: false, code: 400, message: e.toString() };
            }
        },

        /** 停止一个生成式任务
         * @param {String|Number} taskId 任务ID，如果不指定，则默认为当前生成中的任务
         */
        async stop(taskId) {
            let wkId = taskId ? parseInt(taskId) : taskId;
            if (!wkId) {
                if (!this.generateData) {                
                    console.error('Error call stop function: No executing tasks');
                    return;
                }

                wkId = this.generateData.wkId;
            }

            try {
                r = await fetch(`${this.apiBaseUrl}/workProgresses/${wkId}`, {
                        method: 'PATCH',
                        headers: this.headers,
                        body: JSON.stringify({ status: 'abort' })
                    });

                r = await r.json();
                if (r.success) {
                    if (this.generateData.wkId == parseInt(wkId)) {
                        this.generateData.status = 'aborting';
                    }
                }

                return r;

            } catch(e) {
                console.error(e);
                return { success: false, code: 400, message: e.toString() };
            }
        },

        async _checkProgress(flowId) {
            if (!this.executingTasks) {
                console.error('Error call _checkProgress function: No executing tasks');
                return;
            }

            let taskData = this.executingTasks[flowId];
            if (!taskData || !taskData.checkTM || !taskData.wkId) {
                console.error('Error call _checkProgress function: No exists task for workflow id=${flowId}');
                return;
            }

            const updateStorage = () => {
                localStorage.setItem('generateData_task', JSON.stringify(this.executingTasks));
            }
            const checkIsRunning = () => {
                const gen = this.generateData;
                return taskData && gen && taskData.wkId == gen.wkId;
            }
            const _notifyGenEnd = (status, result) => {
                this.$emit('ended', { workflowId: parseInt(flowId), status, result });

                if (checkIsRunning()) this.generateData = null;
                delete this.executingTasks[flowId];

                this.lastGenResult = result; 
                updateStorage();
            }

            taskData.checkTM = null;

            let result = null, genData, r;
            try {
                r = await fetch(`${this.apiBaseUrl}/workProgresses/${taskData.wkId}/event`, {
                        method: 'GET',
                        headers: this.headers,                    
                    });

                r = await r.json();
                genData = this.generateData;
                //console.log(r);
                
                if (r.success) {
                    r = r.data;
                    if (r.cmd) {
                        r = r.data;
                    }
                    
                    taskData = this.executingTasks[flowId];
                    let isRunning = checkIsRunning();

                    if (isRunning) {
                        this.generateData.queueLength = null;
                    }

                    if (isRunning && r.status != genData.status) {
                        this.$emit('statusChanged', { status: r.status, workflowId: parseInt(flowId), taskData, eventData: r.data });
                    }
                    
                    switch(r.status) {
                        case 'wait':
                        case 'started':
                        case 'updating':
                        case 'uploading':
                        case 'downloading':
                            if (isRunning && r.lastProgress) {
                                genData.lastProgress = typeof r.lastProgress == 'string' ? JSON.parse(r.lastProgress) : r.lastProgress;
                            }
                            break;
                            
                        case 'queue':
                            if (isRunning && r.queue != null) {
                                genData.queueLength = r.queue;
                            }
                            break;

                        case 'error':
                        case 'timeout':
                        case 'aborted':
                            if (isRunning)
                                this.generateData = null;
                            
                            return _notifyGenEnd(r.status);
                            
                        case 'ended':
                            let files = 0, apiUrl = this.fileBaseUrl || this.apiBaseUrl;
                            
                            if (r.resourceIds) {
                                result = {
                                    resourceIds: r.resourceIds.split(','),
                                    results: r.results ? JSON.parse(r.results) : null,
                                    ts: Date.now()
                                };

                                result.resources = [];
                                files = result.resourceIds.length;
                                for(const rId of result.resourceIds) {								
                                    result.resources.push({ resourceId: parseInt(rId), url: `${apiUrl}/resources/download/${rId}` });
                                }                            
                            }

                            return _notifyGenEnd('ended', result);
                    }
                }
            } catch(e) {
                console.error(e);                
            }

            if (genData = this.generateData) {
                taskData = this.executingTasks[flowId];
                if (taskData) {
                    taskData.checkTM = setTimeout(() => this._checkProgress(flowId), genData.status == 'aborting' ? 25 : 1000);
                }
            }
        },

        _updateHeaders() {
            const headers = { 'Content-Type': 'application/json', } 

            if (this.token) headers.Authorization = this.token;
            else if (this.apiKey) headers['WXCZ-ACCESS-KEY'] = this.apiKey;
            else return false;

            this.headers = headers;
        }
    }
}
</script>