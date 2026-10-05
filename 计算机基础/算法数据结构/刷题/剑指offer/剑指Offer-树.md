# 剑指Offer-树

# 剑指Offer 7：重建二叉树
输入某二叉树的前序遍历和中序遍历的结果，请重建出该二叉树。假设输入的前序遍历和中序遍历的结果中都不含重复的数字。例如输入前序遍历序列{1,2,4,7,3,5,6,8}和中序遍历序列{4,7,2,1,5,3,8,6}，则重建二叉树并返回。

**思路：**
二叉树的前序遍历顺序是：根节点、左子树、右子树，每个子树的遍历顺序同样满足前序遍历顺序。
二叉树的中序遍历顺序是：左子树、根节点、右子树，每个子树的遍历顺序同样满足中序遍历顺序。
前序遍历是中左右的顺序，中序遍历是左中右的顺序，那么对于{1,2,4,7,3,5,6,8}和{4,7,2,1,5,3,8,6}来说，1是根节点，然后1把中序遍历的序列分割为两部分，“4，7，2”为1的左子树上的节点，“5，3，8，6”为1的右子树上的节点，这样递归的分解下去即可。
![](%E5%89%91%E6%8C%87Offer-%E6%A0%91/72F597D2-9816-4D02-A63A-245D9277F3F0.png)

**实现：**
[剑指Offer(七)：重建二叉树](https://github.com/bryceustc/CodingInterviews/blob/master/ConstructBinaryTree/README.md) (**重要**，前序遍历，中序遍历，后序遍历要掌握)
``` C++
TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
    if (preorder.size() == 0 || inorder.size() == 0) {
        return NULL;
    }
    TreeNode* treeNode = new TreeNode(preorder[0]);
    int mid = distance(begin(inorder), find(inorder.begin(), inorder.end(), preorder[0]));
    vector<int> left_pre(preorder.begin() + 1, preorder.begin() + mid + 1);
    vector<int> right_pre(preorder.begin() + mid + 1, preorder.end());
    vector<int> left_in(inorder.begin(), inorder.begin() + mid);
    vector<int> right_in(inorder.begin() + mid + 1, inorder.end());

    treeNode->left = buildTree(left_pre, left_in);
    treeNode->right = buildTree(right_pre, right_in);
    return treeNode;
}
```

#  剑指Offer 8：二叉树的下一个节点
给定一个二叉树和其中的一个结点，请找出中序遍历顺序的下一个结点并且返回（注意，树中的结点不仅包含左右子结点，同时包含指向父结点的指针）

**思路：**
找中序遍历的下一个节点，本质是考察中序遍历(LDR)，这里最好是画图，分情况讨论。
1. 当该节点存在右子树，下一个节点是右子树最左的节点，也就是右子树中序遍历的第一个节点。
2. 不存在右子树，但是父亲节点的左节点时，下一个节点就是父节点。
3. 不存在右子树，但是父亲节点的右节点时，下一个是沿着父节点往上找第一个父结点是它左子结点的祖先结点。

**实现：**
 [剑指Offer(八：二叉树的下一个节点](https://github.com/bryceustc/CodingInterviews/blob/master/NextNodeInBinaryTrees/README.md) (**重要**，中序遍历，分情况讨论)
``` C++
class Solution {
public:
    TreeLinkNode* GetNext(TreeLinkNode* Node)
    {
        if (Node== NULL)
            return NULL;
        TreeLinkNode* res = NULL;
        // 当前结点有右子树，那么它的下一个结点就是它的右子树中最左子结点
        if (Node->right != NULL)
        {
            TreeLinkNode* pRight = Node->right;
            while(pRight->left!=NULL)
            {
                pRight = pRight->left;
            }
            res = pRight;
        }
        // 当前结点无右子树，则需要找到一个是它父结点的左子树结点的结点
        else if (Node->next!=NULL)
        {
            // 当前结点
            TreeLinkNode* pCur = Node;
            // 父节点
            TreeLinkNode* pNext = Node->next;
            while( pNext != NULL && pNext->right == pCur)
            {
                pCur = pNext;
                pNext = pNext->next;
            }
            res = pNext;
        }
        return res;
    }
};
```

# 剑指Offer 26：树的子结构
输入两棵二叉树A，B，判断B是不是A的子结构。（ps：我们约定空树不是任意一个树的子结构）

**思路：**
结合求2颗树是否相等的问题，用递归解决：
1. B是不是A的子结构 = B是A的左子树的子结构 || B是A的右子树的子结构 || A 和 B 相等。
2. B和A是否相等递归比较value。

**实现：**
 [剑指Offer(二十六)：树的子结构](https://github.com/bryceustc/CodingInterviews/blob/master/SubstructureInTree/README.md) (**重要**，递归遍历思想)
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

#  剑指Offer 27：二叉树的镜像（翻转二叉树）
请完成一个函数，输入一个二叉树，该函数输出它的镜像。
例如输入：
```
     8
   /   \
  6     10
 / \   /  \
5   7 9    11
```
镜像输出：
```
      8
   /    \
  10      6
 /  \    /  \
11   9  7    5
```

**思路1：**
前序递归遍历二叉树，遍历过程中交换左右子树。
**思路2：**
引入队列进行BFS层序迭代遍历，每一次pop动作，pop出每一层的所有节点，对每个节点进行左右子树的交换。

**实现：**
 [剑指Offer(二十七)：二叉树的镜像](https://github.com/bryceustc/CodingInterviews/blob/master/MirrorOfBinaryTree/README.md) (**重要**，翻转二叉树，**递归**)
``` C++
// 1. 递归
class Solution {
public:
    TreeNode* invertTree(TreeNode* root) {
        if (!root) return nullptr;
        TreeNode *p = root->left;
        root->left = root->right;
        root->right = p;
        invertTree(root->left);
        invertTree(root->right);
        return root;
    }
}; 
```
``` C++
// 2.BFS 迭代
#include <deque>
class Solution {
public:
    TreeNode* invertTree(TreeNode* root) {
        if (!root) return root;
        deque<TreeNode*> q;
        q.push_back(root);
        while (!q.empty()) {
            for(int i=0; i<q.size(); i++) {
                TreeNode* n = q.front();
                q.pop_front();
                
                //在这里处理遍历到的每个n节点
                TreeNode* t = n->left;
                n->left = n->right;
                n->right = t;
                
                if (n->left) q.push_back(n->left);
                if (n->right) q.push_back(n->right);
            }
        }
        return root;
    }
};
```

# 剑指Offer 28：对称的二叉树
请实现一个函数，用来判断一棵二叉树是不是对称的。如果一棵二叉树和它的镜像一样，那么它是对称的。
例如，二叉树 [1,2,2,3,4,4,3] 是对称的。
```
    1
   / \
  2   2
 / \ / \
3  4 4  3
```
但是下面这个 [1,2,2,null,3,null,3] 则不是镜像对称的:
```
    1
   / \
  2   2
   \   \
   3    3
```

**思路：**
找出规律：
1. 左子树与右子树对称那么这颗树对称。
2. 左树和右树对称的条件是左树的左孩子与右树的右孩子对称，左树的右孩子与右树的左孩子对称。
3. 递归求解。

**实现**
 [剑指Offer(二十八)：对称的二叉树](https://github.com/bryceustc/CodingInterviews/blob/master/SymmetricalBinaryTree/README.md) (**重要**，递归！！！)
``` C++
class Solution {
public:
    bool isSymmetrical(TreeNode* pRoot)
    {
        bool res = true;
        if (pRoot!=NULL)
        {
            res = helper(pRoot->left,pRoot->right);
        }
        return res;
    }
    bool helper(TreeNode* A, TreeNode* B)
    {
        // 先写递归终止条件
        if (A==NULL && B==NULL)
            return true;
        // 如果其中之一为空，也不是对称的
        if (A==NULL || B==NULL)
            return false;
        // 走到这里二者一定不为空
        if (A->val != B->val)
            return false;
        // 前序遍历
        return helper(A->left,B->right) && helper(A->right,B->left);
    }
};
```

# 剑指Offer 32：从上到下打印二叉树
从上到下打印出二叉树的每个节点，同一层的节点按照从左到右的顺序打印，例如: 给定二叉树: [3,9,20,null,null,15,7]，返回：[3,9,20,15,7]。
```
    3
   / \
  9  20
    /  \
   15   7
```

**思路：**
利用队列进行BFS层序遍历的问题。

**实现**
 [剑指Offer(三十二)：从上往下打印二叉树](https://github.com/bryceustc/CodingInterviews/blob/master/PrintTreeFromTopToBottom/README.md) (**重要**，层序遍历，利用队列实现)
``` C++
class Solution {
public:
    vector<int> PrintFromTopToBottom(TreeNode* root) {
        vector<int> res;
        if (root==NULL)
            return res;
        queue<TreeNode*> q;
        q.push(root);
        while(!q.empty())
        {
            TreeNode* temp = q.front();
            q.pop();
            res.push_back(temp->val);
            if (temp->left)
                q.push(temp->left);
            if (temp->right)
                q.push(temp->right);
        }
        return res;
    }
};

```

# 剑指Offer 32：之字形打印二叉树
请实现一个函数按照之字形顺序打印二叉树，即第一行按照从左到右的顺序打印，第二层按照从右到左的顺序打印，第三行再按照从左到右的顺序打印，其他行以此类推。
例如: 给定二叉树: [3,9,20,null,null,15,7],
```
    3
   / \
  9  20
    /  \
   15   7
```
返回其层次遍历结果：
```
[
  [3],
  [20,9],
  [15,7]
]
```

**思路：**
层序遍历的同时，实现方向相反，可以利用STL容器deque中双端队列push_back(),push_front(),front(),back(),pop(),popfront()实现前取后放，后取前放。

**实现：**
 [剑指Offer(三十二)：之字形打印二叉树](https://github.com/bryceustc/CodingInterviews/blob/master/PrintTreesInZigzag/README.md) (**重要**，利用双端队列实现前取后放，后取前放)
``` C++
class Solution {
public:
    vector<vector<int> > Print(TreeNode* root) {
        vector<vector<int>> res;
        if (root==NULL)
            return res;
        deque<TreeNode*> q;
        q.push_back(root);
        bool zigzag = true; // 从左打印为true
        while (!q.empty())
        {
            int count = q.size();
            vector<int> out;
            TreeNode* node;
            while (count>0)
            {
                if (zigzag) //前取后放：从左向右，所以从前边取，后边放入
                {
                    auto node = q.front();
                    q.pop_front();
                    if (node->left)
                        q.push_back(node->left);
                    if (node->right)
                        q.push_back(node->right);
                } 
                else  // 后取前放：从右向左，从后边取，前边放入
                {
                    node = q.back();
                    q.pop_back();
                    if (node->right)
                        q.push_front(node->right);
                    if (node->left)
                        q.push_front(node->left);
                }
                out.push_back(node->val);
                count--;
            }
            res.push_back(out);
            zigzag = !zigzag;
        }
        return res;
    }
};
```

# 剑指Offer(33)：二叉搜索树的后序遍历序列
输入一个整数数组，判断该数组是不是某二叉搜索树的后序遍历结果。如果是则返回 true，否则返回 false。假设输入的数组的任意两个数字都互不相同。
```
参考以下这颗二叉搜索树：

     5
    / \
   2   6
  / \
 1   3
示例 1：

输入: [1,6,3,2,5]
输出: false
示例 2：

输入: [1,3,2,6,5]
输出: true
```

**思路：**
 后序序列最后一个值为root，二叉搜索树左子树值都比root小，右子树值都比root大。二叉搜索树中，第一个大于root的节点的值在右子树的最左边，同时它也是后序遍历时遍历完左子树后遍历的第一个右子树上的节点。所以：
1. 首先确定root；
2. 遍历序列（除去root结点），第一个大于root的位置，则该位置左边为左子树，右边为右子树；
3. 遍历右子树，若发现有小于root的值，则直接返回false；
4. 分别判断左子树和右子树是否仍是二叉搜索树（即递归步骤1、2、3）。

**实现：**
 [剑指Offer(三十三)：二叉搜索树的后序遍历序列](https://github.com/bryceustc/CodingInterviews/blob/master/SquenceOfBST/README.md) (**重要**，递归的思想，注意下标)
``` C++
class Solution {
public:
    bool VerifySquenceOfBST(vector<int> sequence) {
        bool res = false;
        if (sequence.empty())
            return res;
        int n = sequence.size();
        res = helper(sequence,0,n-1);
        return res;
    }
    bool helper(vector<int> seq, int start, int end)
    {
        if (seq.empty() || start > end)
        {
            return false;
        }
        //根结点
        int root = seq[end];
        
        //在二叉搜索树中左子树的结点小于根结点
        int i = start;
        for (;i<end;i++)
        {
            if (seq[i]>root)
                break;
        }
        
        //在二叉搜索书中右子树的结点大于根结点
        for (int j =i;j<end;j++)
        {
            if (seq[j]<root)
                return false;
        }
        
        //判断左子树是不是二叉搜索树
        bool left = true;
        if (i>start)
        {
            left = helper(seq, start, i-1);
        }
        
        //判断右子树是不是二叉搜索树
        bool right = true;
        if (i < end-1)
        {
            right = helper(seq, i, end-1);
        }
        return left && right;
    }
}
```

#  剑指 Offer 34：二叉树中和为某一值的路径
输入一棵二叉树和一个整数，打印出二叉树中节点值的和为输入整数的所有路径。从树的根节点开始往下一直到叶节点所经过的节点形成一条路径。
**示例:** 给定如下二叉树，以及目标和 sum = 22，
```
              5
             / \
            4   8
           /   / \
          11  13  4
         /  \    / \
        7    2  5   1
```
返回：
```
[
   [5,4,11,2],
   [5,8,4,5]
]
```

**思路：**
此题和[力扣-112路径总和](https://leetcode-cn.com/problems/path-sum/solution/lu-jing-zong-he-by-leetcode/)可以算是同一个题，下面给出的是判断路径总和的思路和代码，区别在于此题优先选用带记忆的DFS迭代方式便于统计出所有的路径。也可以用递归，遍历整棵树，如果当前节点不是叶子，对它的所有孩子节点，递归调用 hasPathSum 函数，其中 sum 值减去当前节点的权值；如果当前节点是叶子，检查 sum 值是否为节点值。

**实现：**
 [剑指Offer(三十四)：二叉树中和为某一值的路径](https://github.com/bryceustc/CodingInterviews/blob/master/PathInTree/README.md) (**重要**，典型带记忆的（&引用）DFS递归方法)
判断路径总和
``` C++
// 1. 递归
class Solution {
public:
    bool hasPathSum(TreeNode* root, int sum) {
        // 题目的测试结果是root == nullptr， sum == 0 也需要返回false
        // 如果不是上面的限制话，可以直接这样写
        // if (!root) return sum == 0;
        if (!root) return false;
        if (!root->left && !root->right) return root->val == sum; 
        return hasPathSum(root->left, sum-root->val) || hasPathSum(root->right, sum-root->val);
    }
};
```
``` C++
// 2. DFS迭代
#include <stack>
using namespace std;

class Solution {
public:
    bool hasPathSum(TreeNode* root, int sum) {
        if (!root) return false;
        typedef pair<TreeNode*, int> Info;
        stack<Info> stack;
        stack.push(Info(root, 0));
        while (!stack.empty())
        {
            auto p = stack.top();
            stack.pop();

            if (p.first) {
                if (p.first->right) stack.push(Info(p.first->right, p.second+p.first->val));
                if (p.first->left) stack.push(Info(p.first->left, p.second+p.first->val));
                stack.push(p);
                stack.push(Info(nullptr, 0));
            } else {
                auto p = stack.top();
                stack.pop();
                // 处理遍历
                if (p.first->left || p.first->right) continue;
                if (p.first->val + p.second == sum) return true;
            }

        }
        return false;
    }
};
```

# 剑指Offer 36：二叉搜索树与双向链表
输入一棵二叉搜索树，将该二叉搜索树转换成一个排序的双向链表。要求不能创建任何新的结点，只能调整树中结点指针的指向。
![](%E5%89%91%E6%8C%87Offer-%E6%A0%91/6971AD58-2C86-406C-B376-348135182A75.png)

**思路：**
1. 基本性质：二叉搜索树的中序遍历为递增序列 。
2. 中序递归访问每个节点各节点 cur，并在访问每个节点时构建cur和前驱节点 
pre的引用指向。
3. 中序遍历完成后，最后构建头节点和尾节点的引用指向即可。

**实现：**
 [剑指Offer(三十六)：二叉搜索树与双向链表](https://github.com/bryceustc/CodingInterviews/blob/master/ConvertBinarySearchTree/README.md) (**重要**，利用中序遍历进行递归)
``` Java
class Solution {
    Node pre, head;
    public Node treeToDoublyList(Node root) {
        if(root == null) return null;
        dfs(root);
        head.left = pre;
        pre.right = head;
        return head;
    }
    void dfs(Node cur) {
        if(cur == null) return;
        dfs(cur.left);
        if(pre != null) pre.right = cur;
        else head = cur;
        cur.left = pre;
        pre = cur;
        dfs(cur.right);
    }
}
```

# 剑指Offer 37：序列化二叉树
请实现两个函数，分别用来序列化和反序列化二叉树。
示例：你可以将以下二叉树，序列化为 "[1,2,3,null,null,4,5]"
```
    1
   / \
  2   3
     / \
    4   5
```
 
**思路：**
1. 无论序列化还是反序列化都需要层序遍历，遍历通过递归或者BFS迭代。
2. 序列化时就是在层序遍历的时候进行字符串拼接处理。
3. 反序列化时先对字符串进行分割，再层序遍历创建节点和指向。

**实现：**
[剑指Offer(三十七)：序列化二叉树](https://github.com/bryceustc/CodingInterviews/blob/master/SerializeBinaryTrees/README.md) (**重要**，前序遍历，递归或者队列迭代)
要用到字符串分割，C++太麻烦了，这里用Java实现。
``` java
public class Codec {
    public String serialize(TreeNode root) {
        if(root == null) return "[]";
        StringBuilder res = new StringBuilder("[");
        Queue<TreeNode> queue = new LinkedList<>() {{ add(root); }};
        while(!queue.isEmpty()) {
            TreeNode node = queue.poll();
            if(node != null) {
                res.append(node.val + ",");
                queue.add(node.left);
                queue.add(node.right);
            }
            else res.append("null,");
        }
        res.deleteCharAt(res.length() - 1);
        res.append("]");
        return res.toString();
    }

    public TreeNode deserialize(String data) {
        if(data.equals("[]")) return null;
        String[] vals = data.substring(1, data.length() - 1).split(",");
        TreeNode root = new TreeNode(Integer.parseInt(vals[0]));
        Queue<TreeNode> queue = new LinkedList<>() {{ add(root); }};
        int i = 1;
        while(!queue.isEmpty()) {
            TreeNode node = queue.poll();
            if(!vals[i].equals("null")) {
                node.left = new TreeNode(Integer.parseInt(vals[i]));
                queue.add(node.left);
            }
            i++;
            if(!vals[i].equals("null")) {
                node.right = new TreeNode(Integer.parseInt(vals[i]));
                queue.add(node.right);
            }
            i++;
        }
        return root;
    }
}
```

# 剑指Offer 54：二叉搜索树的第k个结点
给定一棵二叉搜索树，请找出其中第k大的节点。
示例：输入root = [3,1,4,null,2], k = 1，输出4。
```
**输入:** root = [3,1,4,null,2], k = 1
   3
  / \
 1   4
  \
   2
**输出:** 4
```

**思路：**
本质考察的是中序遍历，二叉搜索树中序遍历先遍历左子树节点的顺序是从小到大，稍作修改先遍历右子树，那么顺序就是从大到小，第k大就是遍历到的第k个节点。

**实现：**
 [剑指Offer(五十四)：二叉搜索树的第k个结点](https://github.com/bryceustc/CodingInterviews/blob/master/KthNodeInBST/README.md) (**重要**，利用递归 或者迭代(堆)进行中序遍历)
``` C++
class Solution {
public:
    int kthLargest(TreeNode* root, int k) {
        if (root == NULL || k<1)
        {
            return 0;
        }
        helper(root, k);
        return res;
    }
    void helper(TreeNode* root, int k)
    {
        if (root==NULL)
            return;
        helper(root->right, k);
        if (++count==k)
        {
            res = root->val;
            return;
        }
        helper(root->left, k);
    }
private:
    int res = 0;
    int count = 0;
};
```

# 剑指Offer 55：二叉树的深度
输入一棵二叉树的根节点，求该树的深度。从根节点到叶节点依次经过的节点（含根、叶节点）形成树的一条路径，最长路径的长度为树的深度。

**思路1：**
递归思想，最大深度=左右子树深度最大的+1。
**思路2：**
BFS遍历思想，最大深度=最大层数，遍历时维护一个最大深度，在弹出每一层上的所有节点时更新深度+1。
**思路3：**
DFS遍历思想，遍历时维护一个最大深度，栈push时计算更新存储的节点和其深度信息，pop时拿出来比较最大的。

**实现：** 
[剑指Offer(五十五)：二叉树的深度](https://github.com/bryceustc/CodingInterviews/blob/master/TreeDepth/README.md) (**重要**，左右子树**dfs递归** 或者bfs迭代层次遍历)
``` C++
// 1. 递归
class Solution {
public:
    int maxDepth(TreeNode* root) {
        if (root == NULL) { 
            return 0;
        } 
        int leftDepath = maxDepth(root->left) + 1;
        int rightDepath = maxDepth(root->right) + 1;
        return leftDepath > rightDepath ? leftDepath : rightDepath;
    }
};

// 2.BFS
#include <deque>
using namespace std;
class Solution {
public:
    int maxDepth(TreeNode* root) {
         if(root==NULL) return 0;
         deque<TreeNode*> q;
         q.push_back(root);
         int deep=0;
         while(!q.empty())
         {
             deep++;
             int num=q.size();
             for(int i=1;i<=num;i++)
             {
                TreeNode* p=q.front();
                q.pop_front();
                if(p->left) q.push_back(p->left);
                if(p->right) q.push_back(p->right);
             }
         }
         return deep;         
    }
};

// 3. DFS
#include <stack>
using namespace std;
class Solution
{
public:
    int maxDepth(TreeNode *root)
    {
         return traversalPre(root);
    }

    int traversalPre(TreeNode* root) {
        typedef pair<TreeNode*,int> Info;
        stack<Info> stack;
    
        if (root != nullptr) {
            stack.push(Info(root, 1));
        }
        int maxDepth = 0;
        while (!stack.empty())
        {
            auto p = stack.top();
            stack.pop();
            if (p.first != NULL) {
                // 前序压栈顺序
                if(p.first->right) stack.push(Info(p.first->right,p.second+1));
                if(p.first->left) stack.push(Info(p.first->left,p.second+1));
                stack.push(p);
                // 在当前节点之前加入一个空节点表示已经访问过了
                stack.push(Info(nullptr, 0));
            } else {
                // 处理遍历到的每一个node的时机
                auto p = stack.top();
                maxDepth = max(maxDepth, p.second);
                stack.pop(); //处理完后出栈
            }
        }
        return maxDepth;
    }
};
```

# 剑指Offer 55：平衡二叉树
输入一棵二叉树的根节点，判断该树是不是平衡二叉树。如果某二叉树中任意节点的左右子树的深度相差不超过1，那么它就是一棵平衡二叉树。

**思路1：**
自顶向下，递归思想，左右子树高度差小于1，并且左右子树也是平衡树。

**思路2：**
自底向上，剪枝+递归思想，对二叉树做后序遍历，从底至顶返回子树最大高度，若判定某子树不是平衡树则 “剪枝” ，直接向上返回。（最优）

**实现：**
 [剑指Offer(五十五)：平衡二叉树](https://github.com/bryceustc/CodingInterviews/blob/master/BalancedBinaryTree/README.md) (**重要**，递归，自顶向下或**自底向上**)
``` C++
// 1.自顶向下，递归思想，时间复杂度 O(Nlog_2 N), 空间(N) 
class Solution {
public:
    bool isBalanced(TreeNode* root) {
        return !root ? true : abs(height(root->left) - height(root->right)) <= 1 && isBalanced(root->left) && isBalanced(root->right);
    }
    int height(TreeNode* node) {
        return !node ? 0 : max(height(node->left), height(node->right)) + 1;
    }
};

// 2.自底向上，剪枝思想，时间复杂度 O(N), 空间(N)
class Solution {
public:
    bool isBalanced(TreeNode* root) {
        return heightInfo(root) != -1;
    }
    int heightInfo(TreeNode* node) {
        if (!node) return 0;
        int left = heightInfo(node->left);
        if (left == -1) return -1;
        int right = heightInfo(node->right);
        if (right == -1) return -1;
        return abs(left -right) < 2 ? max(left, right)+1 : -1;
    }
};
```

