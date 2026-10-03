---
title: FHQ-Treap
date: 2026-10-03 21:20:58
tags: [算法·理论]
---
## 闲话
本人刚学，所以写一篇文章巩固一下。
## 正文

‌FHQ-Treap（无旋 Treap）‌是一种平衡树数据结构，它结合了二叉搜索树和堆的性质，通过分裂和合并操作来保持树的平衡。FHQ-Treap 由范浩强发明，其性能与 Splay 相当，但代码更短且易于理解。
### 首先得有节点

FHQ-Treap 也是一种 BST，所以也是二叉树，自然单个节点需记录左右节点和此节点的值还有这棵子树的节点个数。

为了去重，还要记录与这个点值相同的点个数。

还要记录一个随机值，作为优先级，一颗 Treap 的优先级必然满足堆性质。

代码

```cpp
struct node
{
	node* ch[2];//左右儿子
	int x;//值
	int cnt,sz,rd;//与这个点值相同的点个数，子树节点个数，随机值。
	node(int v)
	{
		x=v;
		cnt=1,sz=1;
		ch[0]=ch[1]=nullptr;
		rd=rand();
	}
	void Update()
	{
		sz=cnt;
		if(ch[0]!=nullptr)
		{
			sz+=ch[0]->sz;
		}
		if(ch[1]!=nullptr)
		{
			sz+=ch[1]->sz;
		}
	}
};

```

其中 Update 函数可以更新子树节点个数。

### 分裂 split

其实这个操作很简单，就是把一颗 Treap，根据一个值 $key$ 分裂成两部分，第一部分所有值小于等于 $key$ ，第二部分所有值大于 $key$。

实现其实也不难。空树只能分成两颗空树。

而对于一颗非空树，如其根节点的值 $\le key$，则由于 BST 性质左子树一定都 $\le key$，只有右子树的节点的值有可能 $>key$。

因此可以再次以 $key$ 分裂右子树，把第一部分作为新的右子树，而第二部分则单独成一棵树。
最后把原来的根作为第一部分，从原来的右子树分裂出的第二部分作为第二部分。

根节点的值 $>key$ 同理，可以自己想一下。
```cpp
pair<node*,node*>split(node* cur,int k)
{
		if(cur==nullptr)
		{
			return {nullptr,nullptr};
		}
		if(cur->x<=k)
		{
			auto tmp=split(cur->ch[1],k);
			cur->ch[1]=tmp.first;
			cur->Update();
			return {cur,tmp.second};
		}
		else
		{
			auto tmp=split(cur->ch[0],k);
			cur->ch[0]=tmp.second;
			cur->Update();
			return {tmp.first,cur};
		}
}
```
### 合并 merge

这个操作稍微好理解点。
说白了就是把两颗 Treap 合并成一颗。

首先给出的两颗树一定是第一颗最大值小于第二颗最大值。因此合并方式至多两种：第一种是把第一颗合并与第二颗的左子树合并，第二种是把第二颗与第一颗的右子树合并。

但到底选哪种也是问题，但我们不是给每个节点记录了一个优先级吗，现在就派上用场了。

如第一棵树根结点的优先级大于第二棵树就执行操作一，否则执行操作二，这样就满足了堆性质，又因为优先级是随机的，所以可以使这颗 Treap 更加平衡。
```cpp
node *merge(node* l,node* r)
	{
		if(l==nullptr&&r==nullptr)
		{
			return nullptr;
		}
		if(l==nullptr&&r!=nullptr)
		{
			return r;
		}
		else if(r==nullptr&&l!=nullptr)
		{
			return l;
		}
		if(l->rd<r->rd)//只是跟描述反过来了而已
		{
			l->ch[1]=merge(l->ch[1],r);
			l->Update();
			return l;
		}
		else
		{
			r->ch[0]=merge(l,r->ch[0]);
			r->Update();
			return r;
		}
	}
```
### 按排名分裂 split_by_rank

也是分裂，只不过是跟据一个排名 $rk$，且要分裂成三颗树。

为了方便，这里介绍一种 STL：[tuple](https://blog.csdn.net/m0_63997099/article/details/136988487)。

分裂出的三棵树分别为：最大排名小于 $rk$ 的，最大排名等于 $rk$ 的，最小排名大于 $rk$ 的。

首先要求根节点的排名，但根节点排名其实就是左子树排名 $+1$。

然后分类讨论，和分裂操作类似，这里不再赘述，有疑惑可看代码。
```cpp

tuple<node*,node*,node*>split_rk(node* cur,int rk)
	{
		if(cur==nullptr)//为空
		{
			return make_tuple(nullptr,nullptr,nullptr);
		}
		int lssz;//左子树节点个数
		if(cur->ch[0]==nullptr)//注意左子树有可能为空
		{
			lssz=0;
		}
		else
		{
			lssz=cur->ch[0]->sz;
		}
		if(rk<=lssz)//根节点排名大于rk
		{
			node *l,*mid,*r;
      //tie绑定函数，有兴趣可以自己看看
			tie(l,mid,r)=split_rk(cur->ch[0],rk);//右子树排名关系确定，分裂左子树
			cur->ch[0]=r;
			cur->Update();
			return make_tuple(l,mid,cur);
		}
		else if(rk<=lssz+cur->cnt)//根节点排名大于等于rk
		{
			node* l=cur->ch[0];
			node* r=cur->ch[1];
			cur->ch[0]=cur->ch[1]=nullptr;
			return make_tuple(l,cur,r);
		}
		else//根节点排名小于rk
		{
			node *l,*mid,*r;//同理
			tie(l,mid,r)=split_rk(cur->ch[1],rk-lssz-cur->cnt);
			cur->ch[1]=l;
			cur->Update();
			return make_tuple(cur,mid,r);
		}
	}
//限于篇幅，所以可能讲的不太清楚，还请各位大佬不要介意。
```
---
### 插入 insert

对于插入一个值为 $v$ 的节点。

先通过两次分裂分裂出三棵树：最大值小于 $v$ 的，等于 $v$ 的，大于 $v$ 的。

如果等于 $v$ 的树为空，则新创建一个值为 $v$ 的点，否则这棵树根的 $cnt$ 加上一，代表又出现了一个和它相同的节点。

最后怎么分裂的怎么合并回来，但如果新创建了节点，一定要合并上！！！
```cpp
void insert(int x)
	{
		auto tmp=split(root,x);
		auto ltr=split(tmp.first,x-1);
		node* nw;
		node* tmp2;
		if(ltr.second==nullptr)
		{
			nw=new node(x);
		}
		else
		{
			ltr.second->cnt++;
			ltr.second->Update();
		}
		if(ltr.second==nullptr)
		{
			tmp2=merge(ltr.first,nw);
		}
		else
		{
			tmp2=merge(ltr.first,ltr.second);
		}
		root=merge(tmp2,tmp.second);
	}
//其实也没多难
```

### 删除 erase
跟插入差不多，仍然要分成三棵树。
但这次变成了减少 $cnt$，减到零就要删除。

**注意这样相同的点只会删除一个**

```cpp
void erase(int x)
	{
		auto tmp=split(root,x);
		auto ltr=split(tmp.first,x-1);
		if(ltr.second->cnt>1)
		{
			ltr.second->cnt--;
			ltr.second->Update();
			ltr.first=merge(ltr.first,ltr.second);
		}
		else
		{
			if(tmp.first==ltr.second)
			{
				tmp.first=nullptr;
			}
			delete ltr.second;//内存释放
			ltr.second=nullptr;
		}
		root=merge(ltr.first,tmp.second);
	}
```

### 根据值查找排名

把树分裂根据值分成两部分，则答案为第一部分节点个数 $+1$。

**记得合并**
```cpp
int query_rk(int x,node* rt)
	{
		auto tmp=split(rt,x-1);
		int rank_v;
		if(tmp.first==nullptr)
		{
			rank_v=1;
		}
		else
		{
			rank_v=tmp.first->sz+1;
		}
		merge(tmp.first,tmp.second);
		return rank_v;
	}
```
### 根据排名查找值

直接按排名分裂即可，就是要合并两次。
```cpp
int query_v(int rk,node* rt)
	{
		node *l,*mid,*r;
		tie(l,mid,r)=split_rk(rt,rk);
		if(mid==nullptr)
		{
			return INF;
		}
		int ans=mid->x;
		root=merge(merge(l,mid),r);
		return ans;
	}
```
### 找第一个小于 $x$ 的节点的值

把整棵树根据 $x-1$ 分裂，查询第一部分排名为第一部分节点个数的值。
```cpp
int query_pre(int x)
	{
		auto tmp=split(root,x-1);
		if(tmp.first==nullptr)
		{
			return -INF;
		}
		T ans=query_v(tmp.first->sz,tmp.first);
		root=merge(tmp.first,tmp.second);
		return ans;
	}
```
### 找第一个大于 $x$ 的节点的值

同理，不再赘述。
```cpp
T query_next(T x)
	{
		auto tmp=split(root,x);
		if(tmp.second==nullptr)
		{
			return INF;
		}
		T ans=query_v(1,tmp.second);
		root=merge(tmp.first,tmp.second);
		return ans;
	}
```

---

## 例题
[P3369	【模板】普通平衡树](https://www.luogu.com.cn/problem/P3369)

就是模板。

代码：

```cpp
#include<bits/stdc++.h>
using namespace std;
using T=int;
T INF=1000000000;
struct node
{
	node* ch[2];
	T x;
	int cnt,sz,rd;
	node(T v)
	{
		x=v;
		cnt=1,sz=1;
		ch[0]=ch[1]=nullptr;
		rd=rand();
	}
	void Update()
	{
		sz=cnt;
		if(ch[0]!=nullptr)
		{
			sz+=ch[0]->sz;
		}
		if(ch[1]!=nullptr)
		{
			sz+=ch[1]->sz;
		}
	}
};
struct Treap
{
	
	node *root;
	pair<node*,node*>split(node* cur,T k)
	{
		if(cur==nullptr)
		{
			return {nullptr,nullptr};
		}
		if(cur->x<=k)
		{
			auto tmp=split(cur->ch[1],k);
			cur->ch[1]=tmp.first;
			cur->Update();
			return {cur,tmp.second};
		}
		else
		{
			auto tmp=split(cur->ch[0],k);
			cur->ch[0]=tmp.second;
			cur->Update();
			return {tmp.first,cur};
		}
	}
	tuple<node*,node*,node*>split_rk(node* cur,int rk)
	{
		if(cur==nullptr)
		{
			return make_tuple(nullptr,nullptr,nullptr);
		}
		int lssz;
		if(cur->ch[0]==nullptr)
		{
			lssz=0;
		}
		else
		{
			lssz=cur->ch[0]->sz;
		}
		if(rk<=lssz)
		{
			node *l,*mid,*r;
			tie(l,mid,r)=split_rk(cur->ch[0],rk);
			cur->ch[0]=r;
			cur->Update();
			return make_tuple(l,mid,cur);
		}
		else if(rk<=lssz+cur->cnt)
		{
			node* l=cur->ch[0];
			node* r=cur->ch[1];
			cur->ch[0]=cur->ch[1]=nullptr;
			return make_tuple(l,cur,r);
		}
		else
		{
			node *l,*mid,*r;
			tie(l,mid,r)=split_rk(cur->ch[1],rk-lssz-cur->cnt);
			cur->ch[1]=l;
			cur->Update();
			return make_tuple(cur,mid,r);
		}
	}
	node *merge(node* l,node* r)
	{
		if(l==nullptr&&r==nullptr)
		{
			return nullptr;
		}
		if(l==nullptr&&r!=nullptr)
		{
			return r;
		}
		else if(r==nullptr&&l!=nullptr)
		{
			return l;
		}
		if(l->rd<r->rd)
		{
			l->ch[1]=merge(l->ch[1],r);
			l->Update();
			return l;
		}
		else
		{
			r->ch[0]=merge(l,r->ch[0]);
			r->Update();
			return r;
		}
	}
	void insert(T x)
	{
		auto tmp=split(root,x);
		auto ltr=split(tmp.first,x-1);
		node* nw;
		node* tmp2;
		if(ltr.second==nullptr)
		{
			nw=new node(x);
		}
		else
		{
			ltr.second->cnt++;
			ltr.second->Update();
		}
		if(ltr.second==nullptr)
		{
			tmp2=merge(ltr.first,nw);
		}
		else
		{
			tmp2=merge(ltr.first,ltr.second);
		}
		root=merge(tmp2,tmp.second);
	}
	void erase(T x)
	{
		auto tmp=split(root,x);
		auto ltr=split(tmp.first,x-1);
		if(ltr.second->cnt>1)
		{
			ltr.second->cnt--;
			ltr.second->Update();
			ltr.first=merge(ltr.first,ltr.second);
		}
		else
		{
			if(tmp.first==ltr.second)
			{
				tmp.first=nullptr;
			}
			delete ltr.second;
			ltr.second=nullptr;
		}
		root=merge(ltr.first,tmp.second);
	}
	T query_rk(T x,node* rt)
	{
		auto tmp=split(rt,x-1);
		int rank_v;
		if(tmp.first==nullptr)
		{
			rank_v=1;
		}
		else
		{
			rank_v=tmp.first->sz+1;
		}
		merge(tmp.first,tmp.second);
		return rank_v;
	}
	T query_v(int rk,node* rt)
	{
		node *l,*mid,*r;
		tie(l,mid,r)=split_rk(rt,rk);
		if(mid==nullptr)
		{
			return INF;
		}
		int ans=mid->x;
		root=merge(merge(l,mid),r);
		return ans;
	}
	T query_pre(T x)
	{
		auto tmp=split(root,x-1);
		if(tmp.first==nullptr)
		{
			return -INF;
		}
		T ans=query_v(tmp.first->sz,tmp.first);
		root=merge(tmp.first,tmp.second);
		return ans;
	}
	T query_next(T x)
	{
		auto tmp=split(root,x);
		if(tmp.second==nullptr)
		{
			return INF;
		}
		T ans=query_v(1,tmp.second);
		root=merge(tmp.first,tmp.second);
		return ans;
	}
};
Treap t;
int main()
{
	srand(time(0));
	//qwq
	int q;
	cin>>q;
	while(q--)
	{
		int opt;
		cin>>opt;
		if(opt==1)
		{
			int x;
			cin>>x;
			t.insert(x);
			/*for(int i=1;i<=t.root->sz;i++)
			{
				cout<<t.query_v(i,t.root)<<",";
			}
			cout<<"\n";*/
		}
		else if(opt==2)
		{
			int x;
			cin>>x;
			t.erase(x);
			/*for(int i=1;i<=t.root->sz;i++)
			{
				cout<<t.query_v(i,t.root)<<",";
			}
			cout<<"\n";*/
		}
		else if(opt==3)
		{
			int x;
			cin>>x;
			cout<<t.query_rk(x,t.root)<<"\n";
		}
		else if(opt==4)
		{
			int x;
			cin>>x;
			cout<<t.query_v(x,t.root)<<"\n";
		}
		else if(opt==5)
		{
			int x;
			cin>>x;
			cout<<t.query_pre(x)<<"\n";
		}
		else if(opt==6)
		{
			int x;
			cin>>x;
			cout<<t.query_next(x)<<"\n";
		}
		
	}
	return 0;
}

```