---
layout: post
title: 队列
description: 描述队列的概念，简单队列的定义和实现。
category: blog
tag: queue,队列
---

## 队列的实现

### c++版本

简单队列的实现我是放在 TVUSignaling 类中的。下面看具体代码。

TVUSignaling.h 文件：

	typedef struct VOIPQNode
	{
	    KSignalingType type;
	    char* data;
	    struct VOIPQNode *next;
	}VOIPQNode;
	
	typedef struct TVUVOIPMessageQueue
	{
	    int size;
	    VOIPQNode *front;
	    VOIPQNode *rear;
	}TVUVOIPMessageQueue;
	
	class TVUSignaling
	{
	public:
	    static TVUSignaling * getInstance();
	    static void closeTVUSignaling();
	    TVUSignaling();
	    ~TVUSignaling();
	    
	    void InitQueue(TVUVOIPMessageQueue *Q);
	    void EnQueue(TVUVOIPMessageQueue *Q, const char* data,int len,KSignalingType type);
	    VOIPQNode *DeQueue(TVUVOIPMessageQueue *Q);
	    VOIPQNode *GetNode(TVUVOIPMessageQueue *Q);
	    void FreeNode(VOIPQNode* Node);
	    void ClearMessageQueue();
	    TVUVOIPMessageQueue *m_messageQueue;
	    
	    
	private:
	    sio::client sclient;
	    void onopen();
	    static TVUSignaling * m_instance;
	    
	    pthread_mutex_t queue_mutex;
	};

TVUSignaling.mm 文件

	TVUSignaling::TVUSignaling()
	{
	    m_messageQueue = (TVUVOIPMessageQueue*)malloc(sizeof(struct TVUVOIPMessageQueue));
	    InitQueue(m_messageQueue);
	    pthread_mutex_init(&queue_mutex, NULL);
	    m_instance = this;
	}
	
	TVUSignaling::~TVUSignaling(){}
	
	TVUSignaling * TVUSignaling::m_instance = NULL;
	TVUSignaling * TVUSignaling::getInstance()
	{
	    if (m_instance == NULL) {
	        m_instance = new TVUSignaling();
	    }
	    return m_instance;
	}
	
	void TVUSignaling::closeTVUSignaling()
	{
	    TVUSignaling::getInstance()->closeConnection();
	}
	
	
	void TVUSignaling::InitQueue(TVUVOIPMessageQueue *Q)
	{
	    Q->size = 0;
	    Q->front = NULL;
	    Q->rear = NULL;
	}
	
	void TVUSignaling::EnQueue(TVUVOIPMessageQueue *Q,const char* data,int len ,KSignalingType type)
	{
	    VOIPQNode* node = (VOIPQNode *)malloc(sizeof(VOIPQNode));
	    node->data = (char *)malloc(len+1);
	    memcpy(node->data, data, len);
	    node->data[len] = 0;
	    node->type = type;
	    node->next = NULL;
	    
	    pthread_mutex_lock(&queue_mutex);
	    
	    if (Q->front == NULL)
	    {
	        Q->front = node;
	        Q->rear = node;
	    }
	    else
	    {
	        Q->rear->next = node;
	        Q->rear = node;
	    }
	    Q->size += 1;
	    pthread_mutex_unlock(&queue_mutex);
	}
	
	VOIPQNode* TVUSignaling::DeQueue(TVUVOIPMessageQueue *Q)
	{
	    VOIPQNode* element = NULL;
	    
	    pthread_mutex_lock(&queue_mutex);
	    
	    element = Q->front;
	    if(element == NULL)
	    {
	        pthread_mutex_unlock(&queue_mutex);
	        return NULL;
	    }
	    
	    Q->front = Q->front->next;
	    Q->size -= 1;
	    pthread_mutex_unlock(&queue_mutex);
	    
	    return element;
	}
	
	void TVUSignaling::FreeNode(VOIPQNode* Node){
	    if(Node != NULL){
	        free(Node->data);
	        free(Node);
	    }
	}
	
	void TVUSignaling::ClearMessageQueue()
	{
	    while (m_messageQueue->size) {
	        VOIPQNode *node = this->DeQueue(m_messageQueue);
	        this->FreeNode(node);
	    }
	}