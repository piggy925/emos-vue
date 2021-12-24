<template>
	<div>
		<el-row :gutter="15">
			<!-- 视频墙 -->
			<el-col :span="19">
				<div class="meeting-container">
					<el-scrollbar height="650px" id="videoListContainer">
						<div class="video-list">
							<div class="video">
								<div class="user">
									<img class="photo" :src="mine.photo" />
									<span class="name">{{ mine.name }}{{ shareStatus ? '（共享中）' : '' }}</span>
								</div>
								<div id="localStream"></div>
							</div>
							<div class="video" v-for="one in memberList">
								<div class="user">
									<img class="photo" :src="one.photo" />
									<span class="name">{{ one.name }}</span>
								</div>
								<div :id="one.id" class="remote-stream" @dblclick="bigVideoHandle(one.id)"></div>
							</div>
						</div>
					</el-scrollbar>
					<div id="videoBig" @dblclick="smallVideoHandle()"></div>
				</div>
				<p class="desc">
					会议过程中，不需要发言的会场要主动将本地会场的MIC关闭，保证会场安静，当需要发言时要及时打开MIC。会议过程中，需要发言讨论时，先打开MIC向主会场提出请求，得到同意后再继续发言，否则请继续保持静音。发言时，要—个人一个人的发言，不要多人同时讲话，因为全向MIC会把所有人的声音混合，远端听到的声音会非常嘈杂，听不清具体说话内容。在会议进行过程中，尽量控制会场噪音，不要在会场中随意走动
				</p>
			</el-col>
			<!-- 用户列表区域 -->
			<el-col :span="5">
				<el-card shadow="never">
					<template #header>
						<div class="card-header"><span>用户列表</span></div>
					</template>
					<el-scrollbar height="555px">
						<ul class="user-list">
							<li v-for="one in userList">
								<img class="mic" src="../assets/trtc/mic.png" />
								<div class="mic-container">
									<img
										class="mic-green"
										:id="'mic-' + one.userId"
										src="../assets/trtc/mic-green.png"
									/>
								</div>
								<span>{{ one.dept }} - {{ one.name }}</span>
							</li>
						</ul>
					</el-scrollbar>
				</el-card>
				<div class="meeting-operate">
					<button :class="meetingStatus ? 'phone-btn-off' : 'phone-btn-on'" @click="phoneHandle"></button>
					<button :class="videoStatus ? 'video-btn-on' : 'video-btn-off'" @click="videoHandle"></button>
					<button :class="micStatus ? 'mic-btn-on' : 'mic-btn-off'" @click="micHandle"></button>
					<button :class="shareStatus ? 'share-btn-on' : 'share-btn-off'" @click="shareHandle"></button>
				</div>
			</el-col>
		</el-row>
	</div>
</template>

<script>
import TRTC from 'trtc-js-sdk';
import $ from 'jquery';

export default {
    data: function () {
        return {
            meetingId: null,
            uuid: null,
            appId: null,
            userSign: null,
            userId: null,
            roomId: null,
            meetingStatus: false,
            videoStatus: true,
            micStatus: true,
            shareStatus: false,
            userList: [], //进入会场的用户列表
            mine: {},
            memberList: [], //会议成员列表
            client: null,
            localStream: null,
            shareStream: null,
            stream: {}, //所有的远端流
            bigVideoUserId: null //大屏显示远端流的用户ID，切换回小屏幕的时候使用
        };
    },
    methods: {
        phoneHandle: function () {
            let that = this;
            TRTC.checkSystemRequirements().then(checkResult => {
                if (!checkResult.result) {
                    that.$alert(
                        "当前浏览器不支持在线视频会议", "提示", {
                            confirmButtonText: "确定"
                        }
                    );
                } else {
                    that.meetingStatus = !that.meetingStatus;
                    if (that.meetingStatus) {
                        // 设置摄像头与麦克风状态
                        that.videoStatus = true;
                        that.micStatus = true;
                        // 设置TRTC日志级别
                        TRTC.Logger.setLogLevel(TRTC.Logger.LogLevel.ERROR);
                        // 生成用户签名
                        that.$http("meeting/searchMyUserSig", "GET", {}, false, resp => {
                            that.appId = resp.appId;
                            that.userSig = resp.userSig;
                            that.userId = resp.userId;
                        });
                        let client = TRTC.createClient({
                            mode: "rtc",
                            sdkAppId: that.appId,
                            userId: that.userId,
                            userSig: that.userSig
                        });
                        that.client = client;
                        // 监听远端流新增事件(用户刚进入会议室时触发)
                        client.on("stream-added", event => {
                            let remoteStream = event.stream;
                            // 订阅远端流
                            client.subscribe(remoteStream);
                            // 从远端流获取远程用户userId
                            let userId = remoteStream.getUserId();

                            // 将新进入会议室的人添加到右侧的在线列表中
                            that.$http("user/searchNameAndDept", "POST", {userId: userId}, true, resp => {
                                that.userList.push({
                                    userId: userId,
                                    userName: resp.userName,
                                    deptName: resp.deptName
                                })
                            });

                            // 将远端流保存到模型层JSON
                            that.stream[userId] = remoteStream;
                        });

                        // 监听远端流订阅成功事件
                        client.on("stream-subscribed", event => {
                            let remoteStream = event.stream;
                            let userId = remoteStream.getUserId();
                            // 用视频墙中的播放视频的用户格子置顶，以覆盖用户信息格子
                            $('#' + userId).css({'z-index': 1});
                            // 在置顶的DIV中播放远端视频
                            remoteStream.play(userId + '');
                        });

                        // 监听远端删除流事件（用户退出会议室）
                        client.on("stream-removed", event => {
                            let remoteStream = event.stream;
                            // 取消订阅该远端流
                            client.unsubscribe(remoteStream);
                            let userId = remoteStream.getUserId();

                            // TODO 在页面右侧的用户列表中删除用户

                            // 停止播放远端流视频、关闭远端流
                            remoteStream.stop();
                            remoteStream.close();
                            // 删除模型层JSON中的远端流对象
                            delete that.stream[userId];
                            // 将视频格子置底，显示用户基本信息
                            $('#' + userId).css({'z-index': '-1'}).html('');
                        });

                        client.join({roomId: that.roomId}).then(() => {
                            // 执行签到
                            that.$http("meeting/updateMeetingPresent", "POST", {meetingId: that.meetingId}, true, resp => {
                                if (resp.rows == 1) {
                                    that.$message({
                                        message: "签到成功",
                                        type: "success",
                                        duration: 1200
                                    });
                                }
                            });

                            // 创建本地流对象
                            let localStream = TRTC.createStream({
                                userId: that.userId + '',
                                audio: true,
                                video: true
                            })
                            that.localStream = localStream;
                            localStream.setVideoProfile("480p");

                            // 将自己添加到右侧用户列表中
                            that.$http("user/searchNameAndDept", "POST", {userId: that.userId}, true, resp => {
                                that.userList.push({
                                    userId: that.userId,
                                    name: resp.name,
                                    dept: resp.dept
                                })
                            });

                            // 初始化本地音视频流
                            localStream.initialize().then(() => {
                                $('#localStream').css({'z-index': '1'});
                                localStream.play('localStream');
                                client.publish().then(() => {
                                    console.log("推送本地流成功")
                                }).catch(error => {
                                    console.log("推送本地流失败: " + error)
                                });
                            }).catch(error => {
                                console.log("初始化本地流失败: " + error)
                            });
                        }).catch(error => {
                            console.log("进入房间失败： " + error);
                        });
                    } else {
                        // TODO 关闭视频会议
                    }
                }
            });
        }
    },
    created: function () {
        let that = this;
        let params = that.$route.params;
        that.meetingId = params.meetingId;
        that.uuid = params.uuid;
        let data = {
            meetingId: that.meetingId
        }
        that.$http("meeting/searchOnlineMeetingMembers", "POST", data, true, resp => {
            let list = resp.list;
            if (list != null && list.length > 0) {
                that.mine = list[0];
                for (let i = 1; i < list.length; i++) {
                    that.memberList.push(list[i]);
                }
            }
        });
        data = {
            uuid: that.uuid
        }
        that.$http("meeting/searchRoomIdByUUID", "POST", data, true, resp => {
            that.roomId = resp.roomId;
        });
    }
}
</script>

<style lang="less" scoped="scoped">
@import url('meeting_video.less');
</style>
