# LeetCode-树

# LeetCode 94、144、145：二叉树的遍历
**思路1：**
递归遍历
``` C++
// 递归实现
void traversal(Node *node){
    if (!node) return;
	  // Todo: 前序处理遍历到的每一个node
    traversal(node->left);
    // Todo: 中序处理遍历到的每一个node
    traversal(node->right);
    // Todo: 后序处理遍历到的每一个node
}
```
**思路2：**
非递归遍历的统一形式，主要思路是模拟递归调用栈。
[力扣](https://leetcode-cn.com/problems/binary-tree-inorder-traversal/solution/wan-quan-mo-fang-di-gui-bu-bian-yi-xing-miao-sha-q/)[更简单的非递归遍历二叉树的方法 - 简书](https://www.jianshu.com/p/49c8cfd07410)
``` C++
void traversal(TreeNode* root) {
    stack<TreeNode*> stack;
    if (root != nullptr) {
        stack.push(root);
    }
    while (!stack.empty())
    {
        TreeNode *p = stack.top();
        stack.pop();
        if (p != NULL) {
            /* 前序压栈顺序
            if(p->right) stack.push(p->right);
            if(p->left) stack.push(p->left);
            stack.push(p);
            // 在当前节点之前加入一个空节点表示已经访问过了
            stack.push(nullptr);
            */

            /* 中序压栈顺序
            if(p->right) stack.push(p->right);
            stack.push(p);
            stack.push(nullptr);
			   // 在当前节点之前加入一个空节点表示已经访问过了
            if(p->left) stack.push(p->left);
            */

            /* 后序压栈顺序
            stack.push(p);
            // 在当前节点之前加入一个空节点表示已经访问过了
            stack.push(nullptr);
            if(p->right) stack.push(p->right);
            if(p->left) stack.push(p->left);
            */
        } else {
            TreeNode *n = stack.top();
            // Todo: 这里处理遍历到的每一个node
            stack.pop(); //处理完后出栈
        }
    }
}
```

# LeetCode 98：验证搜索二叉树
给定一个二叉树，判断其是否是一个有效的二叉搜索树。一个有效的二叉搜索树具有如下特征：
1. 节点的左子树只包含小于当前节点的数。
2. 节点的右子树只包含大于当前节点的数。
3. 所有左子树和右子树自身必须也是二叉搜索树。

**思路：**
基本性质：二叉搜索树中序遍历的结果是节点由小到大。
所以利用中序遍历从最小节点开始访问，引入并维护一个访问当前curr节点时的pre前驱节点，初始为nullptr。如果访问时pre节点不为空并且值大于curr节点值那么说明它不是一个有效的二叉搜索树。

**实现：**
[LeetCode(98):验证搜索二叉树](https://github.com/bryceustc/LeetCode_Note/tree/master/cpp/Validate-Binary-Search-Tree) (**重要**，中序遍历)
``` C++
class Solution {
public:
    TreeNode* pre = NULL;
    bool isValidBST(TreeNode* root) {
        if (root==NULL)
            return true;
        // 访问左子树, 如果左子树为false 返回false
        if (!isValidBST(root->left))
            return false;
        // 访问当前节点：如果当前节点小于等于中序遍历的前一个节点，说明不满足BST，返回 false；否则继续遍历。
        if (pre && root->val <= pre->val)
            return false;
        // 更新前一个节点
        pre = root;
        // 访问右子树
        return isValidBST(root->right);
    }
};
```

# LeetCode 124：二叉树中的最大路径和
给定一个非空二叉树，返回其最大路径和。
本题中，路径被定义为一条从树中任意节点出发，达到任意节点的序列。该路径至少包含一个节点，且不一定经过根节点。
```
输入: [-10,9,20,null,null,15,7]

   -10
   / \
  9  20
    /  \
   15   7

输出: 42
```

**思路：**
1. 计算边的路径，考虑后序从底向上的遍历，计算最大路径，其实就是一个选边的过程。考虑设计一个函数返回经过该节点时选完边后的单边最大贡献值。
2. 空节点的最大贡献值等于 0，非空节点的最大贡献值等于节点值与其子节点中的最大贡献值之和，如果子节点最大贡献值为负的话，就不选择该节点（最大贡献值等于 0）。
3. 全局维护一个最大贡献值，在从下到上后序遍历统计经过每个节点时该节点能提供的最大贡献值的同时，和原来的最大贡献值比较，如果大就更新。

::注意理解更新的时候这里计算的当前贡献最大值=经过左子节点的最大贡献+经过右子节点的最大贡献+本身的值::

::注意理解递归函数的功能是返回选择完后的单边最大贡献值::

**实现：**
 [LeetCode(124):二叉树中的最大路径和](https://github.com/bryceustc/LeetCode_Note/tree/master/cpp/Binary-Tree-Maximum-Path-Sum) (**重要**，递归，max(root, root+left, root+right))
``` C++
class Solution {
public:
    int res = INT_MIN;
    int maxPathSum(TreeNode* root) {
        if (root==NULL) {
            return 0;
        }
        dfs(root);
        return res;
    }
    //后序遍历树，返回经过root的单边最大路径和，并维护整棵树的最大路径和
    int dfs(TreeNode* root) {
        if (root==NULL) {
            return 0;
        }
        //计算左边分支最大值，左边分支如果为负数则不选择
        int left = max(0,dfs(root->left));
        //计算右边分支最大值，右边分支如果为负数则不选择
        int right = max(0,dfs(root->right));
        //由于路径最大的一种可能为left->node->right，而不向root的父结点延伸
        res = max(res,root->val + left + right);
        // 向的父结点返回经过root的单边分支的最大路径和
        return root->val + max(left, right);
    }
    
};
```

# LeetCode 199：二叉树的右视图
给定一棵二叉树，想象自己站在它的右侧，按照从顶部到底部的顺序，返回从右侧所能看到的节点值。 [力扣](https://leetcode-cn.com/problems/binary-tree-right-side-view/)
```
输入: [1,2,3,null,5,null,4]
输出: [1, 3, 4]
解释:

   1            <---
 /   \
2     3         <---
 \     \
  5     4       <---
```

**思路1：**
利用 BFS 进行层次遍历，记录下每层的最后一个元素。
T： O(N)每个节点都入队出队了 1 次。
M：O(N)使用了额外的队列空间。

**思路2：**
利用 DFS 中右左的顺序遍历，这样每层都是最先访问最右边的节点。
T： O(N)每个节点都访问了 1 次。
M：O(N)退化成一条链表，递归时使用的栈空间是 O(N)。

**实现：**
[LeetCode(199):二叉树的右视图](https://github.com/bryceustc/LeetCode_Note/tree/master/cpp/Binary-Tree-Right-Side-View) (**重要**，BFS)
``` C++
// BFS
class Solution {
public:
    vector<int> rightSideView(TreeNode* root) {
        vector<int> res;
        if (root==NULL) return res;
        queue<TreeNode*> q;
        q.push(root);
        while(!q.empty())
        {
            int n = q.size();
            res.push_back(q.front()->val);
            while(n>0)
            {
                TreeNode* temp = q.front();
                q.pop();
                if (temp->right) q.push(temp->right);
                if (temp->left) q.push(temp->left);
                n--;
            }
        }
        return res;
    }
};
```
``` C++
// DFS 
class Solution {
public:
    vector<int> rightSideView(TreeNode* root) {
        vector<int> res;
        dfs(root,0,res);
        return res;
    }
    void dfs(TreeNode* root, int depth, vector<int> &res)
    {
        if (root==NULL) return;
        if (depth==res.size()) // 利用结果res的size等于tree的高度的性质
        {
            res.push_back(root->val);
        }
        dfs(root->right,depth+1, res);
        dfs(root->left, depth+1, res);
    }
};
```

# LeetCode 543：二叉树的直径
给定一棵二叉树，你需要计算它的直径长度。一棵二叉树的直径长度是任意两个结点路径长度中的最大值。这条路径可能穿过也可能不穿过根结点。比如下面这颗树：返回3, 它的长度是路径 [4,2,1,3] 或者[5,2,1,3]。
```
          1
         / \
        2   3
       / \     
      4   5    
```

**思路：**
1. 此题和Leetcode 124：二叉树中的最大路径有些相似，这里换一种自上到下的理解方式。核心还是求最大深度的问题。
2. 维护一个最大直径长度，在计算二叉树某一节点树的最大深度的同时，更新最大直径。

::注意理解更新时当前的最大直径=左子树的深度+右子树的深度::

**实现：**
[LeetCode(543):二叉树的直径](https://github.com/bryceustc/LeetCode_Note/blob/master/cpp/Diameter-Of-Binary-Tree/README.md) (**重要**，利用二叉树的深度公式)
``` C++
class Solution {
public:
    int res = 0;
    int diameterOfBinaryTree(TreeNode* root) {
        if (root==NULL) return 0;
        dfs(root);
        return res;
    }
    // 函数dfs的作用是：找到以root为根节点的二叉树的最大深度
    int dfs(TreeNode* root)
    {
        if (root == NULL) return 0;
        int left = dfs(root->left);
        int right = dfs(root->right);
        res = max(res, left+right);
        return max(left, right) + 1;
    }
};
```

# LeetCode 236：二叉树的最近公共祖先
定一个二叉树, 找到该树中两个指定节点的最近公共祖先。
最近公共祖先的定义为：
对于有根树 T 的两个结点 p、q，最近公共祖先表示为一个结点 x，满足 x 是 p、q 的祖先且 x 的深度尽可能大（一个节点也可以是它自己的祖先）。

**思路：**
DFS后序递归的思路，理解上从顶到底，分别找左右子树上p、q的最近公共祖先，分情况讨论有无情况决定返回哪个，具体：
1. 左右都无返回无。
2. 某一个树有，另一个树无，返回有的那个。
3. 都有说明一边一个，因此 root是他们的最近公共祖先。::（需要再理解）::

**实现：**
[LeetCode(236):二叉树的最近公共祖先](https://github.com/bryceustc/LeetCode_Note/blob/master/cpp/Lowest-Common-Ancestor-Of-A-Binary-Tree/README.md) (**重要**，分清具体情况)
``` C++
class Solution {
public:
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        if (root==NULL ||root==p || root==q) return root;
        TreeNode* left = lowestCommonAncestor(root->left,p,q);
        TreeNode* right = lowestCommonAncestor(root->right,p,q);
        if (left == NULL && right == NULL) return NULL;  // 左右子树同时为空，都不包含p，q，返回NULL
        if (left==NULL) return right;  // 当 left为空 ，right不为空 ：p,q都不在root的左子树中，直接返回 right
        if (right==NULL) return left;  // 与上一条件类似
        return root;  // 同时不为空，说明p，q在左右子树异侧，返回root
    }
};
```

# LeetCode(572):另一个树的子树
给定两个非空二叉树 s 和 t，检验s 中是否包含和 t 具有相同结构和节点值的子树。s 的一个子树包括 s 的一个节点和这个节点的所有子孙。s 也可以看做它自身的一棵子树。

**思路：**
一个树是另一个树的子树，则：
* 要么这两个树相等
* 要么这个树是左树的子树
* 要么这个树是右树的子树
所以采用递归的方式判断，2颗树是否相等的问题也是递归判断。

**实现：**
 [LeetCode(572):另一个树的子树](https://github.com/bryceustc/LeetCode_Note/blob/master/cpp/Subtree-Of-Another-Tree/README.md) (**重要**，递归判断，两棵树是否相同)
``` C++
class Solution {
public:
    bool isSubtree(TreeNode* s, TreeNode* t) {
        if (!t) return true;
        if (!s) return false;
        return isSubtree(s->left,t) || isSubtree(s->right,t) || isSameTree(s,t);
    }
    // 2颗树是否相等
    bool isSameTree(TreeNode* a, TreeNode* b) {
        if (a == nullptr && b == nullptr) return true;
        if (a == nullptr || b == nullptr) return false;
        if (a->val != b->val) return false;
        return isSameTree(a->left,b->left) && isSameTree(a->right,b->right);
    }
};
```
