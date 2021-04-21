---
layout: post
title: "[Java] - 父类/子类重复定义成员变量"
subtitle: 
author: "Dongbo"
header-style: text
hidden: false
tags:
  - java
  - programming
---

今天本来想写 AVL 树练手，但是写基础的二叉搜索树时碰到了Java继承的一个问题，记录一下日后引以为鉴。

首先是定义了以下基础的 `Node `类和 `Tree` 接口：

    public interface Tree {

        void insert(int value);
        int delete(int value);
        void preorderPrint();
        void inorderPrint();
        void BFS();
        void draw();
    }

    public class Node{
        Node left, right;
        int value;

        Node(){}
        Node(int value) {
            this.value = value;
        }
        Node(int value, Node left, Node right){
            this.value = value;
            this.left = left;
            this.right = right;
        }
    }

然后写了一个 `GenericBinaryTree` 类来放一些公共的输出函数


    public class GenericBinaryTree implements Tree{
        Node root;

        @Override
        public void insert(int value) {
        }

        @Override
        public int delete(int value) {
            return 0;
        }

        @Override
        public void preorderPrint() {
        }

        @Override
        public void inorderPrint() {
            System.out.print("inorder:[ ");
            Stack<Node> stack = new Stack<>();
            Node cur = root;
            while(cur != null || !stack.isEmpty()){
                if(cur != null){
                    stack.add(cur);
                    cur = cur.left;
                }
                else{
                    cur = stack.pop();
                    System.out.print(cur.value + " ");
                    cur = cur.right;
                }
            }
            System.out.println("]");
        }

        @Override
        public void BFS() {
            System.out.println("BFS:");
            if(root == null)
                return ;
            Queue<Node> queue = new LinkedList<>();
            queue.add(root);
            while(!queue.isEmpty()){
                for(int i = queue.size(); i > 0; i--){
                    Node cur = queue.poll();
                    System.out.print(cur.value + " ");
                    if(cur.left != null) queue.add(cur.left);
                    if(cur.right != null) queue.add(cur.right);
                }
                System.out.println();
            }
        }

        @Override
        public void draw() {

        }
    }

然后 `BinarySearchTree` 继承 `GenericBinaryTree`

    public class BinarySearchTree extends GenericBinaryTree{
        Node root;

        ......
    }

通过类似下面的测试函数测试时发现，调用 `insert` 函数并没有构建出树，中序遍历和 BFS 访问结果都是空的。

    BinarySearchTree bstree = new BinarySearchTree();
    @Test
    void insert1() {
        int[] arr = {1,3,4,65,5,6,7,20};
        for(int i: arr)
            bstree.insert(i);

        bstree.inorderPrint();
        bstree.BFS();
    }

因为本来没有添加 `GenericBinaryTree` 这个父类的时候，`BinarySearchTree` 的插入是正常的。于是我在父类和子类的 `insert` 函数都加入了输出语句，但是调用的确实是子类的 `insert`。

我再仔细检查了父类和子类的代码，结果发现父类和子类中都定义了 `Node root;`，这样调用 `insert` 时会调用子类重写的 `insert`，数据被插入了子类定义的 `root`；而调用 `BFS` 等没有重写的函数时，访问的时父类中尚未初始化的 `root`，删除子类中的 `root` 程序运行就符合预期了。

    //这里应该有一些关于java继承机制的笔记

写这个简单的二叉搜索树让我深刻的意识到自己代码能力的不足，对于Java以及继承机制的不熟悉，写工程项目的能力也十分欠缺。路漫漫其修远兮，头发掉得还不够多啊。

------------

The End