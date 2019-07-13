<template>
    <div style="height: 100%;">
        <Layout style="height: 100%">
            <Header style="color: white;font-size: 24px;text-align: center">
                成绩查询系统
            </Header>
            <Content style="height: 100%">
                <Modal :fullscreen="true" width="100%" v-model="showImg">
                    <h2 style="text-align: center">原始考卷截图</h2>
                    <span style="font-size: 16px;color: red">看打截图</span>
                    <img width="100%" :src="writingImg"/>
                    <span style="font-size: 16px;color: red">听打截图</span>
                    <img width="100%" :src="listeningImg">
                </Modal>
                <div style="margin: 10px">
                    用户名:<Input v-model="dbName" style="width: 180px;margin-right: 20px"></Input>密码:<Input v-model="dbPassword" style="width: 180px;margin-left: 20px"></Input>
                    考场:<Select v-model="examRoom" style="width:200px">
                                <Option v-for="item in roomList" :value="item.value" :key="item.value">{{ item.label }}</Option>
                             </Select>
                    <Button @click="loadDataFromDB" style="margin-left: 20px" type="success">确认</Button>
                    <Button type="primary" @click="exportGrade">成绩导出</Button>
                </div>
                <div style="background-color: #e8eef7;height: 100%;overflow-y: scroll">
                    <Table ref="gradeTable" border :columns="colHeader" :data="gradeData">

                    </Table>
                </div>
            </Content>
        </Layout>
    </div>
</template>

<script>
    var mysql = require('mysql');



    export default {
        name: "DataExportComp",
        data(){
            return {
                showImg:false,
                examRoom:'',
                writingImg:'',
                listeningImg:'',
                roomList:[
                    {value:1,label:'1号考场'},
                    {value:2,label:'2号考场'},
                    {value:3,label:'3号考场'},
                    {value:4,label:'4号考场'},
                    {value:5,label:'5号考场'},
                    {value:6,label:'6号考场'},
                    {value:7,label:'7号考场'},
                    {value:8,label:'8号考场'},
                    {value:9,label:'9号考场'},
                    {value:10,label:'10号考场'},
                    {value:11,label:'11号考场'},
                    {value:12,label:'12号考场'}
                ],
                colHeader:[
                    {type:'index',width:60,align:'center'},
                    {align:'left',title:'姓名',key:'name',
                    render:(h,params)=>{
                        return h('a',{
                            on:{
                                click:()=>{
                                    this.listeningImg = params.row.listening_img;
                                    this.writingImg = params.row.writing_img;
                                    this.showImg = true;
                                }
                            }
                        },params.row.name);
                    }
                    },
                    {align:'left',title:'考号',key:'examId'},
                    {align:'left',title:'看打正确率',sortable:true,key:'textAccuracy', render:(h,params)=>{
                        // console.log('查看表内容:',params);
                        let acc =  this.computeAccuracyText(params.row);
                        acc = acc*100;
                        acc = parseFloat(acc.toFixed(2));
                        params.row.textAccuracy = acc;
                        return h('div',{style:{
                            color:'#ff0000'
                            }},params.row.textAccuracy+'%');
                    }
                    },
                    {align:'left',sortable:true,title:'听打正确率',key:'audioAccuracy', render:(h,params)=>{
                            // console.log('查看表内容:',params);
                            let acc = this.computeAccuracyAudio(params.row);
                            acc = parseFloat((acc*100).toFixed(2));
                            params.row.audioAccuracy = acc;
                            return h('div',{style:{
                                    color:'#ff0000'
                                }},params.row.audioAccuracy+'%');
                        }},
                    {align:'left',sortable:true,title:'看打速度',key:'textSpeed', render:(h,params)=>{
                            // console.log('查看表内容:',params);
                            params.row.textSpeed = this.computeSpeedText(params.row);
                            return h('div',{style:{
                                    color:'#ff0000'
                                }},[
                                    h('span',params.row.textSpeed),
                                    h('span',{style:{
                                        color:'#8a8a8a','font-size':'10px'
                                        }},'(字/分钟)')
                            ] );
                        }},
                    {align:'left',title:'听打速度',key:'audioSpeed', render:(h,params)=>{
                            // console.log('查看表内容:',params);
                            params.row.audioSpeed = this.computeSpeedAudio(params.row);
                            return h('div',{style:{
                                    color:'#ff0000'
                                }},[
                                h('span',params.row.audioSpeed),
                                h('span',{style:{
                                        color:'#8a8a8a','font-size':'10px'
                                    }},'(字/分钟)')
                            ] );
                        }},
                    {align:'left',title:'看打成绩',key:'textGrade',sortable:true, render:(h,params)=>{
                            // console.log('查看表内容:',params);
                            params.row.textGrade = this.computeGradeText(params.row);
                            return h('div',{style:{
                                    color:'#ff0000'
                                }},params.row.textGrade);
                        }},
                    {align:'left',title:'听打成绩',key:'audioGrade',sortable:true,render:(h,params)=>{
                            // console.log('查看表内容:',params);
                            params.row.audioGrade = this.computeGradeAudio(params.row);
                            return h('div',{style:{
                                    color:'#ff0000'
                                }},params.row.audioGrade);
                        }},
                    {align:'center',title:'总成绩',key:'gradeSum',render:(h,params)=>{
                            console.log('查看表内容:',params);
                            if(params.row.listening_grade == null && params.row.writing_grade==null){
                                params.row.gradeSum = '缺考';
                                return h('div',{style:{
                                        color:'#ff0000'
                                    }},params.row.gradeSum);
                            }
                            params.row.gradeSum = parseFloat(this.computeGradeText(params.row))+parseFloat(this.computeGradeAudio(params.row));
                            params.row.gradeSum = params.row.gradeSum.toFixed(2);
                            return h('div',{style:{
                                    color:'#ff0000'
                                }},params.row.gradeSum);
                        }}

                ],
                dbName:'',dbPassword:'',
                gradeData:[],connection:null
            };
        },
        methods:{
            computeAccuracyText(row){
                if (row.writing_grade == "NaN" || row.writing_grade == null){
                    return 0;
                }
                let accuracy = row.writing_grade / row.writing_num;
                return accuracy;

            },
            computeAccuracyAudio(row){
                if (row.listening_grade == "NaN" || row.listening_grade == null){
                    return 0;
                }
                let accuracy = row.listening_grade / row.listening_num;
                return accuracy;
            },
            computeSpeedText(row){
                let speed = row.writing_grade / row.writing_time;
                speed = (speed*60).toFixed(2);
                return speed;
            },
            computeSpeedAudio(row){

                let speed = row.listening_grade / row.listening_time;
                speed = (speed*60).toFixed(2);
                return speed;
            },
            computeGradeText(row){
                if (row.writing_grade == null ||row.writing_grade=="NaN"){
                    return 0;
                }
                let grade = row.writing_grade/row.writing_time * 60;
                //速度分
                let speed_grade = 0;
                if (grade <= 50){
                    speed_grade = 15 * (grade/50);
                }
                else {
                    speed_grade = 15 + (grade - 50)/50*10;
                    if (speed_grade>25){
                        speed_grade = 25;
                    }
                }
                //正确率分
                let accuracy = this.computeAccuracyText(row)*25;
                //总分
                let sum = parseFloat((accuracy+speed_grade).toFixed(2));
                return sum;
            },
            computeGradeAudio(row){
                if (row.writing_grade == null ||row.writing_grade=="NaN"){
                    return 0;
                }
                let grade = row.listening_grade/row.listening_time * 60;
                //速度分
                let speed_grade = 0;
                if (grade <= 50){
                    speed_grade = 15 * (grade/50);
                }
                else {
                    speed_grade = 15 + (grade - 50)/50*10;
                    if (speed_grade>25){
                        speed_grade = 25;
                    }
                }
                //正确率分
                let accuracy = this.computeAccuracyAudio(row)*25;
                //总分
                let sum = parseFloat((accuracy+speed_grade).toFixed(2));
                return sum;
            },
            exportGrade(){
                this.$refs.gradeTable.exportCsv({
                    filename: this.examRoom+'考场 考生成绩信息',original:false
                });
            },
            async loadDataFromDB(){
                if (this.dbName.trim() == '' || this.dbPassword.trim()==''){
                    this.$Modal.error({content:'请输入数据库账号和密码'});
                    return;
                }
                console.log('执行完成');
                this.connection = mysql.createConnection({
                    host     : 'localhost',
                    user     : this.dbName,
                    password : this.dbPassword,
                    database : 'exam'
                });
                let conn = this.connection;
                try{
                    console.log('准备连接...');
                    let err =await new Promise(resolve => {
                        conn.connect({},(error)=>{
                            resolve(error);
                        });
                    });
                    if (err!=null){
                        throw new Error(err);
                    }
                }
                catch (e) {
                    console.log('发生异常');
                    this.$Modal.error({title:'提示',content:'数据库账号或密码错误'});
                    return ;
                }
                this.$Message.info('连接成功');
                let sql = 'select u.username as name,u.mobile as examId,g.* from tb_user u left join grade_user g on u.mobile = g.user_id';
                conn.query(sql,(err,results,field)=>{
                    if (err!=null){
                        this.$Message.info('查询数据库时发生错误');
                        return;
                    }
                    console.log(results);
                    this.gradeData = results;
                });
            }
        }
    }
</script>

<style scoped>

</style>
