<template>
    <el-row class="row">
        <el-row class="row" style="height: 50px;line-height: 50px;text-align: center">
            <el-input style="width: 90%" placeholder="请输入你要筛选的用户名" v-model="searchName"></el-input>
        </el-row>
        <el-collapse>
            <template v-for="(item,index) in arrFilter">
                <el-collapse-item class="hover" :key="item._id">
                    <template slot="title">
                            <span style="font-size: 15px">
                                {{item.name+"("+item.users.length+")"}}
                            </span>&nbsp;&nbsp;&nbsp;
                        <el-dropdown v-if="session.teamRole==0 || session.teamRole==2">
                            <el-button type="text" icon="setting" class="el-dropdown-link" @click.stop="">
                            </el-button>
                            <el-dropdown-menu slot="dropdown">
                                <el-dropdown-item @click.native="inviteUser(item)">邀请用户</el-dropdown-item>
                                <el-dropdown-item @click.native="renameGroup(item)">重命名</el-dropdown-item>
                                <el-dropdown-item @click.native="removeGroup(item,index)">删除</el-dropdown-item>
                            </el-dropdown-menu>
                        </el-dropdown>
                    </template>
                    <template v-for="(item1,index1) in item.users">
                        <el-row class="row" style="height: 40px;line-height: 40px;text-align: center" :key="item1.user._id">
                            <el-col class="col" :span="4">
                                <img v-proxy="item1.user.photo" style="width: 30px;height: 30px; border-radius:50%;vertical-align: middle">
                            </el-col>
                            <el-col class="col" :span="10">
                                {{item1.user.name}}
                            </el-col>
                            <el-col class="col" :span="6">
                                <template v-if="item1.role!=2">
                                    <el-select v-if="session.teamRole==0 || session.teamRole==2" v-model="item1.role" @input="changeRole(item1)">
                                        <el-option :value="0" label="团队管理员"></el-option>
                                        <el-option :value="1" label="团队成员"></el-option>
                                    </el-select>
                                    <span v-else>
                                        {{item1.role==0?'团队管理员':'团队成员'}}
                                    </span>
                                </template>
                                <span v-else>
                                    团队所有者
                                </span>
                            </el-col>
                            <el-col class="col" :span="4">
                                <el-dropdown v-if="session.teamRole==0 || session.teamRole==2">
                                    <el-button type="text"  class="el-dropdown-link" @click.stop="">
                                        操作
                                    </el-button>
                                    <el-dropdown-menu slot="dropdown">
                                        <el-dropdown-item @click.native="moveUser(item1,index1,item)">移动</el-dropdown-item>
                                        <el-dropdown-item @click.native="removeUser(item1,index1,item)" v-if="item1.role!=2">删除</el-dropdown-item>
                                    </el-dropdown-menu>
                                </el-dropdown>
                            </el-col>
                        </el-row>
                    </template>
                </el-collapse-item>
            </template>
        </el-collapse>
        <el-dialog title="选择部门" v-model="showGroup" size="small">
            <el-row class="row" style="height: 30px;line-height: 30px;text-align: center">
                <el-select v-model="selectGroup" style="width: 50%">
                    <el-option :label="item.name" :value="item.id" v-for="item in arrGroup" :key="item.id"></el-option>
                </el-select>
            </el-row>
            <span slot="footer" class="dialog-footer">
            <el-button type="primary" @click="handleMoveUser" :loading="movePending">确 定</el-button>
        </span>
        </el-dialog>
    </el-row>
</template>

<script>
    var proxyImg=require("../../director/proxyImg");
    var sessionChange=require("../../mixins/session");
    module.exports={
        props:["arr"],
        data:function () {
            return {
                searchName:"",
                showGroup:false,
                movePending:false,
                moveUserId:"",
                selectGroup:"",
                moveUserIndex:-1,
                moveUserParent:null
            }
        },
        mixins:[sessionChange],
        directives:{
            proxy:proxyImg
        },
        computed:{
            arrFilter:function () {
                if(!this.searchName)
                {
                    return this.arr;
                }
                var arr=[];
                var _this=this;
                this.arr.forEach(function (obj) {
                    var objCopy=$.clone(obj);
                    objCopy.users=objCopy.users.filter(function (obj) {
                        if(obj.user.name.toLowerCase().indexOf(_this.searchName.toLowerCase())>-1)
                        {
                            return true;
                        }
                        else
                        {
                            return false;
                        }
                    })
                    if(objCopy.users.length>0)
                    {
                        arr.push(objCopy);
                    }
                })
                return arr;
            },
            arrGroup:function () {
                return this.arr.map(function (obj) {
                    return {
                        name:obj.name,
                        id:obj._id
                    }
                })
            }
        },
        methods:{
            inviteUser:function (item) {
                var _this=this;
                $.input("请输入邀请的用户名",function (val) {
                    if(!val.value)
                    {
                        $.tip("请输入邀请的用户名",0);
                        return;
                    }
                    $.startHud();
                    net.put("/team/pulluser",{
                        id:session.get("teamId"),
                        user:val.value,
                        group:item._id
                    }).then(function (data) {
                        $.stopHud();
                        if(data.code==200)
                        {
                            $.notify("邀请已发送，等待用户响应",1);
                        }
                        else
                        {
                            $.notify(data.msg,0);
                        }
                    })
                })
            },
            renameGroup:function (item) {
                var _this=this;
                $.input("请输入部门名称",function (val) {
                    if(!val.value)
                    {
                        $.tip("请输入部门名称",0);
                        return;
                    }
                    $.startHud();
                    net.post("/team/group",{
                        id:session.get("teamId"),
                        name:val.value,
                        group:item._id
                    }).then(function (data) {
                        $.stopHud();
                        if(data.code==200)
                        {
                            $.notify("重命名成功",1);
                            item.name=val.value;
                        }
                        else
                        {
                            $.notify(data.msg,0);
                        }
                    })
                })
            },
            removeGroup:function (item,index) {
                var _this=this;
                if(item.users.length>0)
                {
                    $.tip("请在删除部门前清空部门成员!",0);
                    return;
                }
                else
                {
                    $.confirm("是否删除该部门",function () {
                        $.startHud();
                        net.delete("/team/group",{
                            id:session.get("teamId"),
                            group:item._id
                        }).then(function (data) {
                            $.stopHud();
                            if(data.code==200)
                            {
                                $.notify("删除成功",1);
                                _this.arr.splice(index,1);
                            }
                            else
                            {
                                $.notify(data.msg,0);
                            }
                        })
                    })
                }
            },
            changeRole:function (item) {
                net.put("/team/userrole",{
                    id:session.get("teamId"),
                    user:JSON.stringify([{
                        user:item.user._id,
                        role:item.role
                    }])
                }).then(function (data) {
                    if(data.code==200)
                    {
                        $.notify("设置成功",1);
                    }
                    else
                    {
                        $.notify(data.msg,0);
                    }
                })
            },
            moveUser:function (item,index,parent) {
                this.showGroup=true;
                this.moveUserId=item.user._id
                this.moveUserIndex=index;
                this.moveUserParent=parent;
                this.selectGroup=parent._id
            },
            removeUser:function (item,index,parent) {
                var _this=this;
                $.confirm("是否确定从团队中删除该用户，团队中的项目也会一并删除该用户",function () {
                    $.startHud();
                    net.delete("/team/projectuser",{
                        id:session.get("teamId"),
                        user:item.user._id
                    }).then(function (data) {
                        if(data.code==200)
                        {
                            if(data.data.length>0)
                            {
                                $.stopHud();
                                var child=$.showBox(_this,"teamUserOwner",{
                                    arr:data.data,
                                    user:item.user._id
                                },"team/component")
                                child.$on("remove",function () {
                                    parent.users.splice(index,1);
                                    _this.$store.state.event.$emit("updateTeamUser",-1);
                                })
                            }
                            else
                            {
                                net.delete("/team/user",{
                                    id:session.get("teamId"),
                                    user:item.user._id
                                }).then(function (data) {
                                    $.stopHud();
                                    if(data.code==200)
                                    {
                                        $.notify("删除成功",1);
                                        parent.users.splice(index,1);
                                        _this.$store.state.event.$emit("updateTeamUser",-1);
                                    }
                                    else
                                    {
                                        $.notify(data.msg,0);
                                    }
                                })
                            }
                        }
                        else
                        {
                            $.stopHud();
                            $.notify(data.msg,0);
                        }
                    })
                })
            },
            handleMoveUser:function () {
                if(!this.selectGroup)
                {
                    $.tip("请选择部门",0);
                    return;
                }
                else if(this.selectGroup==this.moveUserParent._id)
                {
                    $.tip("请不要移动到同一个部门",0);
                    return;
                }
                var _this=this;
                this.movePending=true;
                net.put("/team/moveuser",{
                    id:session.get("teamId"),
                    user:this.moveUserId,
                    group:this.selectGroup
                }).then(function (data) {
                    _this.movePending=false;
                    if(data.code==200)
                    {
                        $.notify("移动成功",1);
                        _this.showGroup=false;
                        _this.moveUserParent.users.splice(_this.moveUserIndex,1);
                        var group;
                        _this.arr.forEach(function (obj) {
                            if(obj._id==_this.selectGroup)
                            {
                                group=obj;
                            }
                        })
                        group.users.push(data.data);
                        group.users.sort(function (obj1,obj2) {
                            return obj1.user.name>obj2.user.name
                        })
                    }
                    else
                    {
                        $.notify(data.msg,0);
                    }
                })
            }
        }
    }
</script>