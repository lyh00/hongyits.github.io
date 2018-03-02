---
layout: post
title: "利用websocket做商城聊天室"
date: 2018-2-7 
description: "一一一一一"
tag: 个人学习
---
### 最近在做毕设的时候，要做一个买家跟卖家之间的聊天功能。考虑到二者之间要来回互动，所以用activeMQ的话要配置太多监听器，感觉很麻烦，正好websocket面试经常用到，所以选择websocket加zdialog实现这个功能。

## 首先，实现websocket的有2种方式，[实现websocket的两种方式](http://blog.csdn.net/zzhao114/article/details/60154017)，因为我框架是SSM，因为对spring不太熟，所以虽然不难，但是途中配置还是出现了几个问题，浪费了点时间，所以记录下来，以提高以后再用到的效率。

## 创建一个配置文件，在这边添加拦截器跟handler。注意：这边路径名后缀要跟拦截的后缀一样，因为这个东西耽搁了有一会。
	
	 // Component注解告诉SpringMVC该类是一个SpringIOC容器下管理的类
	 // 其实@Controller, @Service, @Repository是@Component的细化
	@Component
	@EnableWebSocket
	public class MyWebSocketConfig implements WebSocketConfigurer {

	@Override
	public void registerWebSocketHandlers(WebSocketHandlerRegistry webSocketHandlerRegistry) {
		// 添加websocket处理器，添加握手拦截器
		MyWebSocketHandler handler = getHandler();
		System.out.println(handler);
		webSocketHandlerRegistry.addHandler(handler, "/myHandler.html").addInterceptors(new MyHandShakeInterceptor());
		// 添加websocket处理器，添加握手拦截器
		webSocketHandlerRegistry.addHandler(handler, "/ws/sockjs").addInterceptors(new MyHandShakeInterceptor()).withSockJS();
	}

	private MyWebSocketHandler getHandler() {
		return new MyWebSocketHandler();
	}
	}


## 配置一个拦截器
	//websocket握手拦截器 拦截握手前，握手后的两个切面
	public class MyHandShakeInterceptor extends HttpSessionHandshakeInterceptor {
	@Override
	public boolean beforeHandshake(ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse,
			WebSocketHandler webSocketHandler, Map<String, Object> map) throws Exception {
		// 获得登录拦截器设置的用户信息
		TbUser user = (TbUser) ((ServletServerHttpRequest) serverHttpRequest).getServletRequest().getAttribute("user");
		String isAdmin = ((ServletServerHttpRequest) serverHttpRequest).getServletRequest().getParameter("type");
		System.out.println("Websocket:用户[id:" + user + "]已简历连接");
		if (serverHttpRequest instanceof ServletServerHttpRequest) {
			if (StringUtils.isNoneBlank(isAdmin)) {// 是管理员
				map.put("uid", "0");
			} else {
				// 不是管理员
				if (user != null) {
					map.put("uid", user.getId());
					System.out.println("用户ID" + user.getId() + "被加入");
				} else {
					System.out.println("user为空");
					return false;
				}

			}
		}
		return true;
	}

	@Override
	public void afterHandshake(ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse,
			WebSocketHandler webSocketHandler, Exception arg3) {

	}

	}
## 配置一个handler
	@Component
		public class MyWebSocketHandler extends TextWebSocketHandler {
	// 当MyWebSocketHandler类被加载时就会创建该Map，随类而生
	public static final Map<Integer, WebSocketSession> userSocketSessionMap;

	static {
		userSocketSessionMap = new HashMap<Integer, WebSocketSession>();
	}

	// 握手实现连接后
	@Override
	public void afterConnectionEstablished(WebSocketSession session) throws Exception {
		String uidString = session.getAttributes().get("uid").toString();
		Integer uid = Integer.valueOf(uidString);
		if (userSocketSessionMap.get("uid") == null) {
			userSocketSessionMap.put(uid, session);
		}

	}

	// 发送信息前的处理
	@Override
	public void handleMessage(WebSocketSession webSocketSession, WebSocketMessage<?> webSocketMessage)
			throws Exception {
		System.out.println("handleMessage");
		if (webSocketMessage.getPayloadLength() == 0)
			return;
		// 得到Socket通道中的数据并转化为Message对象
		System.out.println("后台得到的消息:" + webSocketMessage.getPayload().toString());
		 Message message = JsonUtils.jsonToPojo(webSocketMessage.getPayload().toString(),
		 Message.class);
		// 发送Socket信息
		 sendMessageToUser(message.getToId(),new TextMessage(new
		 GsonBuilder().setDateFormat("yyyy-MM-d HH:mm:ss").create().toJson(message)));

	}

	/**
	 * 在此刷新页面就相当于断开WebSocket连接,原本在静态变量userSocketSessionMap中的
	 * WebSocketSession会变成关闭状态(close)，但是刷新后的第二次连接服务器创建的
	 * 新WebSocketSession(open状态)又不会加入到userSocketSessionMap中,所以这样就无法发送消息
	 * 因此应当在关闭连接这个切面增加去除userSocketSessionMap中当前处于close状态的WebSocketSession，
	 * 让新创建的WebSocketSession(open状态)可以加入到userSocketSessionMap中
	 * 
	 * @param webSocketSession
	 * @param closeStatus
	 * @throws Exception
	 */
	@Override
	public void afterConnectionClosed(WebSocketSession webSocketSession, CloseStatus closeStatus) throws Exception {
		System.out.println("WebSocket:" + webSocketSession.getAttributes().get("uid") + "close connection");
		Iterator<Map.Entry<Integer, WebSocketSession>> iterator = userSocketSessionMap.entrySet().iterator();
		while (iterator.hasNext()) {
			Map.Entry<Integer, WebSocketSession> entry = iterator.next();
			if (entry.getValue().getAttributes().get("uid") == webSocketSession.getAttributes().get("uid")) {
				userSocketSessionMap.remove(webSocketSession.getAttributes().get("uid"));
				System.out.println("WebSocket in staticMap:" + webSocketSession.getAttributes().get("uid") + "removed");
			}
		}

	}

	@Override
	public void handleTransportError(WebSocketSession arg0, Throwable arg1) throws Exception {

	}

	@Override
	public boolean supportsPartialMessages() {
		return false;
	}

	// 发送信息的实现
	public void sendMessageToUser(int toId, TextMessage message) throws IOException {
		WebSocketSession session = userSocketSessionMap.get(toId);
		System.out.println(toId);
		if (session != null && session.isOpen()) {
			session.sendMessage(message);
		}
	}

}

## 在xml配置中拦截器，并且添加xml规范(xmlns:websocket及xsi:schemaLocation)。
	<!-- 配置好处理器 -->
	<bean id="websocketHandler" class="com.huanghongyuan.front.webSocket.MyWebSocketHandler" />
	<!-- 配置拦截器 -->
	<websocket:handlers>
		<websocket:mapping path="/myHandler.html" handler="websocketHandler" /> 
		<websocket:handshake-interceptors>
			<bean class="com.huanghongyuan.front.webSocket.MyHandShakeInterceptor" />
		</websocket:handshake-interceptors>
	</websocket:handlers>

## 添加websocket的jar包，注意：websocket-api跟websocket会有冲突，取websocket即可。
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-messaging</artifactId>
			<version>4.2.4.RELEASE</version>
		</dependency>
		<dependency>
			<groupId>com.fasterxml.jackson.core</groupId>
			<artifactId>jackson-core</artifactId>
			<version>2.3.0</version>
		</dependency>
		<dependency>
			<groupId>com.fasterxml.jackson.core</groupId>
			<artifactId>jackson-databind</artifactId>
			<version>2.3.0</version>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-websocket</artifactId>
			<version>4.0.5.RELEASE</version>
		</dependency>
		<dependency>
			<groupId>com.google.code.gson</groupId>
			<artifactId>gson</artifactId>
			<version>2.3.1</version>
		</dependency>
## 前端
	var data = {}; //新建data对象，并规定属性名与相应的值
	data['fromId'] = null;
	data['fromName'] = null;
	data['toId'] = 0; //客户端默认发给管理员.
	data['messageText'] = "";
	//开启websocket
	var ws = new WebSocket("ws://" + window.location.host + "/myHandler.html");
	ws.onopen = function(event) {
		console.log("连接成功");
		console.log(event);
	};
	ws.onerror = function(event) {
		console.log("连接失败");
		console.log(event);
	};
	ws.onclose = function(event) {
		console.log("Socket连接断开");
		console.log(event);
	};
	ws.onmessage = function(event) {
		//接受来自服务器的消息
		var data = JSON.parse(event.data);
		var getMessage = data.messageText;
		//显示收到的消息
		var appendMsg = "";
		appendMsg += '<tr>';
		appendMsg += '<td style="text-align: left;color: red;">' + getMessage + '</td>';
		appendMsg += '</tr>';
		$("#charRoomShow2").append(appendMsg);

	};
	//发送信息
	function sendMsg() {
		var user = JSON.parse($("#user").val()); //登录的user对象
		var messageText = $("#chatRoomInput").val();
		data['fromId'] = user.id;
		data['fromName'] = user.username;
		data['messageText'] = messageText;
		//显示发送的消息
		var appendMsg = "";
		appendMsg += '<tr>';
		appendMsg += '<td style="text-align: right;color: blue;">' + messageText + '</td>';
		appendMsg += '</tr>';
		$("#charRoomShow2").append(appendMsg);
		ws.send(JSON.stringify(data)); //将对象封装成JSON后发送至服务器
	}
