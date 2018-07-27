值得学习记录的博客：http://liweizhaolili.blog.163.com/blog/static/162307442013621101937980/

Large Terrain Rendering：https://blog.csdn.net/qq_29523119/article/details/56017155

一、了解LOD是什么
要渲染数公里的地形，您需要采用各种详细程度（LOD）技术之一。LOD是一组技术，他们共同的目标就是减少远处
地形上的多边形数量和复杂着色器使用。如果您尝试渲染大量地形而不减少任何细节，您很快发现普通显卡无法以
合理的速度渲染它。然而，即使是简单的LOD实现也可以让您绘制大量的地形，同事保持令人印象深刻的帧速率。

有许多不同的LOD算法可以动态地减少远程地形的细节。然而，当你接近遥远的地形的时候，他们中的大多数都会
造成可怕的地形“爆裂”效果，这些效果会让人眼花缭乱。同样，这些相同的动态算法通常会遇到地形边缘无法正确
对其的问题，从而导致地形出现明显的裂缝。为了避免LOD算法的两个常见问题，我们将覆盖一种使用预先构建的
地形方法，该方法没有这些问题，我们将要使用的方法称为基于节点的LOD。

二、了解基于节点的LOD（Node Based LOD）

基于节点的LOD是一种将整个地形均匀分解为多个节点的方法。每个节点大小相同。例如128*128。现在，这种方法
最重要的是每个节点都有很多个质量等级。例如，节点可能具有低、中、高质量版本，该版本已预先在terrain编
辑器中构建。然后，在我们渲染该节点的时候，我们确定他与摄像机的距离，并从中选择要渲染的质量版本。如果
它靠近相机然后我们渲染高质量版本，如果它超过一定范围但在最大渲染范围内，那么我们回执低质量版本。您当然
可以拥有尽可能多的等级，但通常三到四个品质就足够了。

基于节点的LOD的下一部分是当摄像机在地形上移动的时候，我们使用后台线程来加载和卸载节点。大多数视频卡没有
足够内存来容纳数公里的多个质量地形节点，因此需要加载/卸载。为了防止帧速率被中断，我们需要使用后台线程来
进行加载。例如，当摄像机靠近中等质量节点时，它需要加载更高质量的版本。一旦加载它，然后切换到渲染高质量
的版本。最后它卸载了之前使用的中等质量版本。


请注意，在DirectX 11中，您不应创建和释放顶点和索引缓冲区以执行地形节点的加载和卸载。这样做会导致视频卡锁定并导致内存抖动。
您应该为每个质量级别使用预初始化的动态顶点缓冲区数组。重用这些缓冲区而不是释放和重新创建。由于每个质量级别的缓冲区大小相同，
因此您只需使用Map函数覆盖旧数据即可。例如，如果您有九个高质量节点，那么您应该有一个由十个顶点和索引缓冲区组成的阵列，
专用于高质量节点。当您需要加载新的高质量节点时，可以使用第十个未使用的顶点/索引缓冲区并将地形节点数据映射到其中
，并为节点指定一个指针。然后标记前一个高质量顶点和索引缓冲区，该缓冲区不再仅仅使用布尔标志呈现为未使用。

对于每个质量级别，我们还有一个着色器。用于近距离地形的超复杂着色器绝不应用于远距离地形。还尝试在渲染时按照着色器排序，并尝试
最小化常量缓冲区更新以获得最佳帧率。

基于节点的LOD的最后一部分是我们根据摄像机的位置渲染节点。摄像机当前所在的节点及其周围的八个节点必须呈现为高质量。
这使我们能够始终保持高质量环绕相机的区域，并且还可以让我们有足够的时间在相机移动时加载更多高质量的节点。
请注意，高频效果（如法线贴图纹理）不会显示在距离中，因此不需要超过9个高质量地形节点。

围绕9个高质量节点的下16个节点呈现为中等质量。您可以以中等质量执行超过16个（例如围绕这16个节点的接下来的24个节点），
但这是完全灵活的，并且由您决定一旦测试完毕后由您决定。

https://blog.csdn.net/u012234115/article/details/48055399

一、同样要了解LOD技术

多分辨率网格简化技术/细节层次模拟技术（LOD技术），引入“分治”的思想，根据地形的不同复杂度和人眼观察
地形的特点，对地形的不同区域采取不同细节的描述和绘制 。采用LOD技术绘制地形，在不降低表现效果的前提
下，可以尽量减少三角形的数量，以提高图形绘制效率，实现地形的事实交互可视化。

基于四叉树的连续细节层次模型的构建算法，是基于规则网格的具有代表性的研究成果之一。算法的“连续”特性
包含了3个意识：

1.帧更新的时候，地形表面保持连续性，即时间的连续性；

2.不同分辨率的地形之间保持连续性，没有裂缝，即空间连续性。

3.算法的实时构网能力很强，以保持较高的频率刷新率。

建立第一个地形

http://www.doc88.com/p-0058542481623.html

https://blog.csdn.net/gentle_wolf/article/details/6067757

参考了很多相关LOD技术的论文，也了解了LOD技术的基本宗旨以及简单的内容。
我们首先不依赖高度图信息来进行建立一个地形。

代码记录：

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
//我们的二维网格图左上角第一个位子为（0，0），编码为0；
[RequireComponent(typeof(MeshFilter))]
[RequireComponent(typeof(MeshRenderer))]
class TreeNode
{ 
    //Id换算到二维网格图中的时候的换算公式：
    //x=Id/N;
    //y=Id-(Id/N)*N;
    public int Id;

    //当前点控制的层级
    //级别换算到真正地形上的大小公式：
    //Mianji=(N*N)/(Pow(4,Level));
    
    public int Level;
    //左上角
    public TreeNode Child1;
    //右上角
    public TreeNode Child2;
    //左下角
    public TreeNode Child3;
    //右上角
    public TreeNode Child4;
    //构造函数
    public TreeNode(int num,TreeNode a,TreeNode b,TreeNode c,TreeNode d)
    {
        Id = num;

        Child1 = a;
        Child2 = b;
        Child3 = c;
        Child4 = d;
    }
    //构造函数
    public TreeNode(int num)
    {
        Id = num;
   
        Child1 = null;
        Child2 = null;
        Child3 = null;
        Child4 = null;
    }
}
//我们首先建立一个纯平坦地形，根据摄像头高度之类的相关内容，搞出来一个地形先看着。
public class LowerTerrain : MonoBehaviour {

    public const float C = 5f;
    static public int N = 65;
    static public int LOD = 5;
    private bool[,] IsOpen=new bool[N+1,N+1];
    TreeNode root;

	// Use this for initialization
	void Start () {
        print("yes");
        gameObject.AddComponent<MeshFilter>();
        gameObject.AddComponent<MeshRenderer>();
       // DrawACircleTest();
        BuildBaseTerrain();
        BuildHugeTerrain();
    }
	
	// Update is called once per frame
	void Update () {
        BuildBaseTerrain();
        BuildHugeTerrain();
    }
    public void DrawACircleTest()
    {
        Mesh mesh = gameObject.GetComponent<MeshFilter>().mesh;
        mesh.Clear();
        int n = 361;
        Vector3[] vp3 = new Vector3[n];
        int[] array = new int[(n - 1) * 3];
        //每次计算点移动的度数
        float Move = 1f;
        //当前转到的度数
        float Now = 0;
        //半径
        float R = 3;
        //设定圆心
        vp3[0] = new Vector3(0, 0, 0);
        for (int i = 1; i < n; i++)
        {
            //接下来算圆上每个点的位子；
            //每次转动角度为Move；
            float x = vp3[0].x + R * Mathf.Cos(Now * Mathf.PI / 180f);
            float y = vp3[0].y + R * Mathf.Sin(Now * Mathf.PI / 180f);
            float z = 0;
            vp3[i] = new Vector3(x, y, z);
            Now += Move;
        }
        for (int i = 0; i < n - 1; i++)
        {
            array[i * 3] = 0;
            array[i * 3 + 1] = i + 2;
            array[i * 3 + 2] = i + 1;
            if (array[i * 3] > n - 1) array[i * 3] -= n - 1;
            if (array[i * 3 + 1] > n - 1) array[i * 3 + 1] -= n - 1;
            if (array[i * 3 + 2] > n - 1) array[i * 3 + 2] -= n - 1;
        }
        mesh.vertices = vp3;
        mesh.triangles = array;
    }
    public void BuildBaseTerrain()
    {
        for (int i = 0; i < N; i++)
        {
            for (int j = 0; j < N; j++)
            {
                IsOpen[i, j] = false;
            }
        }
        int All = (N)*(N);
        GameObject camera = GameObject.Find("Main Camera");
        //得到摄像机镜头的位子。
        Vector3 Pos = camera.transform.position;
        print(Pos.x + " " + Pos.y + " " + Pos.z);
        //然后Bfs得到这颗树。
        //同时能够得到IsOpen的情况
        

        Queue<TreeNode> S=new Queue<TreeNode>();
        S.Clear();
        TreeNode temp = new TreeNode(All / 2);
        temp.Level = 0;
        S.Enqueue(temp);
        root = temp;
        while (S.Count>0)
        {
            TreeNode Now = S.Dequeue();
            float L = GetDistance(Pos, new Vector3(Now.Id / N, 0, (float)(Now.Id - (int)(Now.Id / N) * N)));
            float D = (float)N*N/Mathf.Pow(4, Now.Level);
            
            float c = L / D;
            //print(L +  "  " + D);
            if (c < C && Now.Level + 1 <= LOD) 
            {
                int nx = Now.Id / N;
                int ny = Now.Id - (int)(Now.Id / N) * N;
                int cx = (int)Mathf.Pow(2,LOD-Now.Level-1);
                int cy = (int)Mathf.Pow(2,LOD-Now.Level-1);
                TreeNode C1 = new TreeNode(N * (nx - cx) + (ny - cy));
                TreeNode C2 = new TreeNode(N * (nx - cx) + (ny + cy));
                TreeNode C3 = new TreeNode(N * (nx + cx) + (ny - cy));
                TreeNode C4 = new TreeNode(N * (nx + cx) + (ny + cy));
                C1.Level = Now.Level + 1;
                C2.Level = Now.Level + 1;
                C3.Level = Now.Level + 1;
                C4.Level = Now.Level + 1;
                Now.Child1 = C1;
                Now.Child2 = C2;
                Now.Child3 = C3;
                Now.Child4 = C4;
                print("继续分裂:"+nx + " " + ny);
                IsOpen[nx, ny] = true;
                S.Enqueue(C1);
                S.Enqueue(C2);
                S.Enqueue(C3);
                S.Enqueue(C4);
            }
            else
            {
                int nx = Now.Id / N;
                int ny = Now.Id - (int)(Now.Id / N) * N;
                IsOpen[nx, ny] = false;
            }
        }
        /*
        for(int i=0;i<N;i++)
        {
            print(IsOpen[i, 0] + " " + IsOpen[i, 1] + " " + IsOpen[i, 2] + " " + IsOpen[i, 3] + " " + IsOpen[i, 4] + " " + IsOpen[i, 5]);
        }
        */
    }
    public void BuildHugeTerrain()
    {
        int cnt = 0;
        int cnt1 = 0;
        int cnt2 = 0;
        List<Vector3> vp3 = new List<Vector3>();
        List<Vector2> UV = new List<Vector2>();
        List<int> array = new List<int>();
        Queue<TreeNode> S = new Queue<TreeNode>();
        S.Enqueue(root);
        while(S.Count>0)
        {
            TreeNode Now = S.Dequeue();
            if(Now.Child1!=null&&Now.Child2!=null&&Now.Child3!=null&&Now.Child4!=null)
            {
                TreeNode c1 = Now.Child1;
                TreeNode c2 = Now.Child2;
                TreeNode c3 = Now.Child3;
                TreeNode c4 = Now.Child4;
                S.Enqueue(c1);
                S.Enqueue(c2);
                S.Enqueue(c3);
                S.Enqueue(c4);
            }
            else
            {
                int nx = Now.Id / N;
                int ny = Now.Id - (int)(Now.Id / N) * N;
                int cx = (int)Mathf.Pow(2, LOD - Now.Level);
                int cy = (int)Mathf.Pow(2, LOD - Now.Level);
                Vector3 one = new Vector3(nx - cx, 0, ny - cy);
                Vector3 two = new Vector3(nx - cx, 0,ny + cy);
                Vector3 three = new Vector3(nx + cx, 0,ny - cy);
                Vector3 fore = new Vector3(nx + cx,0 ,ny + cy);
                vp3.Add(one);
                vp3.Add(two);
                vp3.Add(three);
                vp3.Add(fore);
                UV.Add(new Vector2(one.x, one.z));
                UV.Add(new Vector2(two.x, two.z));
                UV.Add(new Vector2(three.x, three.z));
                UV.Add(new Vector2(fore.x, fore.z));
                cnt += 4;cnt1 += 4;
                array.Add(cnt1 - 4);
                array.Add(cnt1 - 3);
                array.Add(cnt1 - 1);

                array.Add(cnt1 - 1);
                array.Add(cnt1 - 3);
                array.Add(cnt1 - 4);

                array.Add(cnt1 - 4);
                array.Add(cnt1 - 1);
                array.Add(cnt1 - 2);

                array.Add(cnt1 - 2);
                array.Add(cnt1 - 1);
                array.Add(cnt1 - 4);

                cnt2 += 12;

            }
        }
        Vector2[] uv = new Vector2[cnt];
        Vector3[] vpp = new Vector3[cnt1];
        int[] arrray = new int[cnt2];
        for (int i = 0; i < cnt; i++) uv[i] = UV[i];
        for (int i = 0; i < cnt1; i++) vpp[i] = vp3[i];
        for (int i = 0; i < cnt2; i++) arrray[i] = array[i];
        Mesh mesh = gameObject.GetComponent<MeshFilter>().mesh;
        mesh.uv = uv;
        mesh.vertices = vpp;
        mesh.triangles = arrray;
    }
    public float GetDistance(Vector3 A,Vector3 B)
    {
        float Dis;
        //print("A:" + A.x + " " + A.y + " " + A.z);
        //print("B:" + B.x + " " + B.y + " " + B.z);
        Dis = Mathf.Sqrt((A.x - B.x) * (A.x - B.x) + (A.y - B.y) * (A.y - B.y) + (A.z - B.z) * (A.z - B.z));
        //print("Dis:"+Dis);
        return Dis;
    }
}
```

效果图：

![](https://i.loli.net/2018/07/26/5b5983126d19a.jpg)

代码记录：

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
//我们的二维网格图左上角第一个位子为（0，0），编码为0；
[RequireComponent(typeof(MeshFilter))]
[RequireComponent(typeof(MeshRenderer))]
class TreeNode
{ 
    //Id换算到二维网格图中的时候的换算公式：
    //x=Id/N;
    //y=Id-(Id/N)*N;
    public int Id;

    //当前点控制的层级
    //级别换算到真正地形上的大小公式：
    //Mianji=(N*N)/(Pow(4,Level));
    
    public int Level;
    //左上角
    public TreeNode Child1;
    //右上角
    public TreeNode Child2;
    //左下角
    public TreeNode Child3;
    //右上角
    public TreeNode Child4;
    //构造函数
    public TreeNode(int num,TreeNode a,TreeNode b,TreeNode c,TreeNode d)
    {
        Id = num;

        Child1 = a;
        Child2 = b;
        Child3 = c;
        Child4 = d;
    }
    //构造函数
    public TreeNode(int num)
    {
        Id = num;
   
        Child1 = null;
        Child2 = null;
        Child3 = null;
        Child4 = null;
    }
}
//我们首先建立一个纯平坦地形，根据摄像头高度之类的相关内容，搞出来一个地形先看着。
public class LowerTerrain : MonoBehaviour {
    //8193 12 0.12
    //1025 9 5 存在断层
    //129 6 5


    //从灰度图中提取一个地形
    public Texture2D heightmaps;

    public Material tempmaterial;
    private float tim = 0;
    public const float C = 5f;

    //长宽
    static public int N = 1025;
    //高
    static public int H = 150;
    static public int LOD = 9;
    private bool[,] IsOpen=new bool[N+1,N+1];
    TreeNode root;

	// Use this for initialization
	void Start () {
        print("yes");
        gameObject.AddComponent<MeshFilter>();
        gameObject.AddComponent<MeshRenderer>();
       // DrawACircleTest();
        BuildBaseTerrain();
        BuildHugeTerrain();
    }
	
	// Update is called once per frame
	void Update () {
        //BuildBaseTerrain();
        //BuildHugeTerrain();
    }
    public void DrawACircleTest()
    {
        Mesh mesh = gameObject.GetComponent<MeshFilter>().mesh;
        mesh.Clear();
        int n = 361;
        Vector3[] vp3 = new Vector3[n];
        int[] array = new int[(n - 1) * 3];
        //每次计算点移动的度数
        float Move = 1f;
        //当前转到的度数
        float Now = 0;
        //半径
        float R = 3;
        //设定圆心
        vp3[0] = new Vector3(0, 0, 0);
        for (int i = 1; i < n; i++)
        {
            //接下来算圆上每个点的位子；
            //每次转动角度为Move；
            float x = vp3[0].x + R * Mathf.Cos(Now * Mathf.PI / 180f);
            float y = vp3[0].y + R * Mathf.Sin(Now * Mathf.PI / 180f);
            float z = 0;
            vp3[i] = new Vector3(x, y, z);
            Now += Move;
        }
        for (int i = 0; i < n - 1; i++)
        {
            array[i * 3] = 0;
            array[i * 3 + 1] = i + 2;
            array[i * 3 + 2] = i + 1;
            if (array[i * 3] > n - 1) array[i * 3] -= n - 1;
            if (array[i * 3 + 1] > n - 1) array[i * 3 + 1] -= n - 1;
            if (array[i * 3 + 2] > n - 1) array[i * 3 + 2] -= n - 1;
        }
        mesh.vertices = vp3;
        mesh.triangles = array;
    }
    public void BuildBaseTerrain()
    {
        for (int i = 0; i < N; i++)
        {
            for (int j = 0; j < N; j++)
            {
                IsOpen[i, j] = false;
            }
        }
        int All = (N)*(N);
        GameObject camera = GameObject.Find("Main Camera");
        //得到摄像机镜头的位子。
        Vector3 Pos = camera.transform.position;
        print(Pos.x + " " + Pos.y + " " + Pos.z);
        //然后Bfs得到这颗树。
        //同时能够得到IsOpen的情况
        

        Queue<TreeNode> S=new Queue<TreeNode>();
        S.Clear();
        TreeNode temp = new TreeNode(All / 2);
        temp.Level = 0;
        S.Enqueue(temp);
        root = temp;
        while (S.Count>0)
        {
            TreeNode Now = S.Dequeue();

            int nx = Now.Id / N;
            int ny = Now.Id - (int)(Now.Id / N) * N;
            float Bx = (float)(1.0f / (float)N);
            float Bz = (float)(1.0f / (float)N);
            float L = GetDistance(Pos, new Vector3(Now.Id / N, GetCellHeight(heightmaps,new Vector2(nx*Bx,ny*Bz)), (float)(Now.Id - (int)(Now.Id / N) * N)));
            float D = (float)N*N/Mathf.Pow(4, Now.Level);
            
            float c = L / D;
            //print(L +  "  " + D);
            if (c < C && Now.Level + 1 <= LOD) 
            {

                int cx = (int)Mathf.Pow(2,LOD-Now.Level-1);
                int cy = (int)Mathf.Pow(2,LOD-Now.Level-1);
                TreeNode C1 = new TreeNode(N * (nx - cx) + (ny - cy));
                TreeNode C2 = new TreeNode(N * (nx - cx) + (ny + cy));
                TreeNode C3 = new TreeNode(N * (nx + cx) + (ny - cy));
                TreeNode C4 = new TreeNode(N * (nx + cx) + (ny + cy));
                C1.Level = Now.Level + 1;
                C2.Level = Now.Level + 1;
                C3.Level = Now.Level + 1;
                C4.Level = Now.Level + 1;
                Now.Child1 = C1;
                Now.Child2 = C2;
                Now.Child3 = C3;
                Now.Child4 = C4;
                print("继续分裂:"+nx + " " + ny);
                IsOpen[nx, ny] = true;
                S.Enqueue(C1);
                S.Enqueue(C2);
                S.Enqueue(C3);
                S.Enqueue(C4);
            }
            else
            {
                IsOpen[nx, ny] = false;
            }
        }
        /*
        for(int i=0;i<N;i++)
        {
            print(IsOpen[i, 0] + " " + IsOpen[i, 1] + " " + IsOpen[i, 2] + " " + IsOpen[i, 3] + " " + IsOpen[i, 4] + " " + IsOpen[i, 5]);
        }
        */
    }
    public void BuildHugeTerrain()
    {
        int cnt = 0;
        int cnt1 = 0;
        int cnt2 = 0;
        List<Vector3> vp3 = new List<Vector3>();
        List<Vector2> UV = new List<Vector2>();
        List<int> array = new List<int>();
        Queue<TreeNode> S = new Queue<TreeNode>();
        S.Enqueue(root);
        while(S.Count>0)
        {
            TreeNode Now = S.Dequeue();
            if(Now.Child1!=null&&Now.Child2!=null&&Now.Child3!=null&&Now.Child4!=null)
            {
                TreeNode c1 = Now.Child1;
                TreeNode c2 = Now.Child2;
                TreeNode c3 = Now.Child3;
                TreeNode c4 = Now.Child4;
                S.Enqueue(c1);
                S.Enqueue(c2);
                S.Enqueue(c3);
                S.Enqueue(c4);
            }
            else
            {
                //我们是根据灰度图的高度图来计算点，需要提前做好UV点坐标，才能够得到每个点的真正高度
                //这个点值得注意一下。、
                int nx = Now.Id / N;
                int ny = Now.Id - (int)(Now.Id / N) * N;
                int cx = (int)Mathf.Pow(2, LOD - Now.Level);
                int cy = (int)Mathf.Pow(2, LOD - Now.Level);
                Vector3 one = new Vector3(nx - cx, 0, ny - cy);
                Vector3 two = new Vector3(nx - cx, 0,ny + cy);
                Vector3 three = new Vector3(nx + cx, 0,ny - cy);
                Vector3 fore = new Vector3(nx + cx,0 ,ny + cy);
                float Bx = (float)(1.0f / (float)N);
                float Bz = (float)(1.0f / (float)N);
                UV.Add(new Vector2(one.x * Bx, one.z * Bz));
                UV.Add(new Vector2(two.x * Bx, two.z * Bz));
                UV.Add(new Vector2(three.x * Bx, three.z * Bz));
                UV.Add(new Vector2(fore.x * Bx, fore.z * Bz));
                
                one.y = GetCellHeight(heightmaps, new Vector2(one.x * Bx, one.z * Bz));
                two.y = GetCellHeight(heightmaps, new Vector2(two.x * Bx, two.z * Bz));
                three.y = GetCellHeight(heightmaps, new Vector2(three.x * Bx, three.z * Bz));
                fore.y = GetCellHeight(heightmaps, new Vector2(fore.x * Bx, fore.z * Bz));
                
                
                vp3.Add(one);
                vp3.Add(two);
                vp3.Add(three);
                vp3.Add(fore);

                cnt += 4;cnt1 += 4;
                array.Add(cnt1 - 4);
                array.Add(cnt1 - 3);
                array.Add(cnt1 - 1);
                /*
                array.Add(cnt1 - 1);
                array.Add(cnt1 - 3);
                array.Add(cnt1 - 4);
                */
                array.Add(cnt1 - 4);
                array.Add(cnt1 - 1);
                array.Add(cnt1 - 2);
                /*
                array.Add(cnt1 - 2);
                array.Add(cnt1 - 1);
                array.Add(cnt1 - 4);
                */
                cnt2 += 6;

            }
        }
        print(cnt + "   "+ cnt1+ "    " + cnt2);
        Vector2[] uv = new Vector2[cnt];
        Vector3[] vpp = new Vector3[cnt1];
        int[] arrray = new int[cnt2];
        for (int i = 0; i < cnt; i++) uv[i] = UV[i];
        for (int i = 0; i < cnt1; i++) vpp[i] = vp3[i];
        for (int i = 0; i < cnt2; i++) arrray[i] = array[i];
        Mesh mesh = gameObject.GetComponent<MeshFilter>().mesh;
        mesh.vertices = vpp;
        mesh.uv = uv;
        mesh.triangles = arrray;
        gameObject.GetComponent<Renderer>().material = tempmaterial;
        /*
        for (int i = 0; i < cnt; i++) 
        {
            print(vpp[i].x + " " + vpp[i].z + "Uv：" + uv[i].x + " " +uv[i].y);
        }
        */
        print(cnt2/3);
    }
    
    public float GetCellHeight(Texture2D map,Vector2 uv)
    {
        // 如果贴图不为空
        if (map != null)
        {
            Color c = GetCellColor(map, uv);
            float gray = c.grayscale;
            float h = H * gray;
            return h;
        }
        else return 0;
    }
    public Color GetCellColor(Texture2D map,Vector2 uv)
    {
        Color color = map.GetPixel(Mathf.FloorToInt(map.width * uv.x), Mathf.FloorToInt(map.height * uv.y));
        return color;
    }
    
    public float GetDistance(Vector3 A,Vector3 B)
    {
        float Dis;
        //print("A:" + A.x + " " + A.y + " " + A.z);
        //print("B:" + B.x + " " + B.y + " " + B.z);
        Dis = Mathf.Sqrt((A.x - B.x) * (A.x - B.x) + (A.y - B.y) * (A.y - B.y) + (A.z - B.z) * (A.z - B.z));
        //print("Dis:"+Dis);
        return Dis;
    }
}
```

现在阶段遇到的问题：

1.当N=1025&&H=150&&LOD=9&&C=5f的情况下，出现断层。

2.显示效果极其不理想。

断层问题稍后处理，显示效果不理想，其实原因是要重置一下法线和范围。
两个语句能够解决这个问题：
```C#
//重置法线
mesh.RecalculateNormals();
//重置范围
mesh.RecalculateBounds();
```

过程代码记录：

```C#

using System.Collections;
using System.Collections.Generic;
using UnityEngine;
//我们的二维网格图左上角第一个位子为（0，0），编码为0；
[RequireComponent(typeof(MeshFilter))]
[RequireComponent(typeof(MeshRenderer))]
class TreeNode
{
    //Id换算到二维网格图中的时候的换算公式：
    //x=Id/N;
    //y=Id-(Id/N)*N;
    public int Id;

    //当前点控制的层级
    //级别换算到真正地形上的大小公式：
    //Mianji=(N*N)/(Pow(4,Level));

    public int Level;
    //左上角
    public TreeNode Child1;
    //右上角
    public TreeNode Child2;
    //左下角
    public TreeNode Child3;
    //右上角
    public TreeNode Child4;
    //构造函数
    public TreeNode(int num, TreeNode a, TreeNode b, TreeNode c, TreeNode d)
    {
        Id = num;

        Child1 = a;
        Child2 = b;
        Child3 = c;
        Child4 = d;
    }
    //构造函数
    public TreeNode(int num)
    {
        Id = num;

        Child1 = null;
        Child2 = null;
        Child3 = null;
        Child4 = null;
    }
}
//我们首先建立一个纯平坦地形，根据摄像头高度之类的相关内容，搞出来一个地形先看着。
public class LowerTerrain : MonoBehaviour
{
    //8193 12 0.12
    //1025 9 5 存在断层
    //129 6 5


    //从灰度图中提取一个地形
    public Texture2D heightmaps;

    public Material tempmaterial;
    private float tim = 0;
    public float C = 0.02f;

    //长宽
    public int N = 1025;
    //高
    public int H = 150;
    public int LOD = 9;
    private bool[,] IsOpen = new bool[15000, 15000];
    TreeNode root;

    // Use this for initialization
    void Start()
    {
        print("yes");
        gameObject.AddComponent<MeshFilter>();
        gameObject.AddComponent<MeshRenderer>();
        // DrawACircleTest();
        BuildBaseTerrain();
        BuildHugeTerrain();
    }

    // Update is called once per frame
    void Update()
    {
        /*
        if (tim <= 5f)
        {
            tim += Time.deltaTime;
        }
        else
        {
            tim = 0;
            BuildBaseTerrain();
            BuildHugeTerrain();
        }
        */
    }
    public void DrawACircleTest()
    {
        Mesh mesh = gameObject.GetComponent<MeshFilter>().mesh;
        mesh.Clear();
        int n = 361;
        Vector3[] vp3 = new Vector3[n];
        int[] array = new int[(n - 1) * 3];
        //每次计算点移动的度数
        float Move = 1f;
        //当前转到的度数
        float Now = 0;
        //半径
        float R = 3;
        //设定圆心
        vp3[0] = new Vector3(0, 0, 0);
        for (int i = 1; i < n; i++)
        {
            //接下来算圆上每个点的位子；
            //每次转动角度为Move；
            float x = vp3[0].x + R * Mathf.Cos(Now * Mathf.PI / 180f);
            float y = vp3[0].y + R * Mathf.Sin(Now * Mathf.PI / 180f);
            float z = 0;
            vp3[i] = new Vector3(x, y, z);
            Now += Move;
        }
        for (int i = 0; i < n - 1; i++)
        {
            array[i * 3] = 0;
            array[i * 3 + 1] = i + 2;
            array[i * 3 + 2] = i + 1;
            if (array[i * 3] > n - 1) array[i * 3] -= n - 1;
            if (array[i * 3 + 1] > n - 1) array[i * 3 + 1] -= n - 1;
            if (array[i * 3 + 2] > n - 1) array[i * 3 + 2] -= n - 1;
        }
        mesh.vertices = vp3;
        mesh.triangles = array;
    }
    public void BuildBaseTerrain()
    {
        for (int i = 0; i < N; i++)
        {
            for (int j = 0; j < N; j++)
            {
                IsOpen[i, j] = false;
            }
        }
        int All = (N) * (N);
        GameObject camera = GameObject.Find("Main Camera");
        //得到摄像机镜头的位子。
        Vector3 Pos = camera.transform.position;
        print(Pos.x + " " + Pos.y + " " + Pos.z);
        //然后Bfs得到这颗树。
        //同时能够得到IsOpen的情况


        Queue<TreeNode> S = new Queue<TreeNode>();
        S.Clear();
        TreeNode temp = new TreeNode(All / 2);
        temp.Level = 0;
        S.Enqueue(temp);
        root = temp;
        while (S.Count > 0)
        {
            TreeNode Now = S.Dequeue();

            int nx = Now.Id / N;
            int ny = Now.Id - (int)(Now.Id / N) * N;
            float Bx = (float)(1.0f / (float)N);
            float Bz = (float)(1.0f / (float)N);
            float L = GetDistance(Pos, new Vector3(Now.Id / N, GetCellHeight(heightmaps, new Vector2(nx * Bx, ny * Bz)), (float)(Now.Id - (int)(Now.Id / N) * N)));
            float D = (float)N * N / Mathf.Pow(4, Now.Level);

            float c = L / D;
            //print(L +  "  " + D);
            if (c < C && Now.Level + 1 <= LOD)
            {

                int cx = (int)Mathf.Pow(2, LOD - Now.Level - 1);
                int cy = (int)Mathf.Pow(2, LOD - Now.Level - 1);
                TreeNode C1 = new TreeNode(N * (nx - cx) + (ny - cy));
                TreeNode C2 = new TreeNode(N * (nx - cx) + (ny + cy));
                TreeNode C3 = new TreeNode(N * (nx + cx) + (ny - cy));
                TreeNode C4 = new TreeNode(N * (nx + cx) + (ny + cy));
                C1.Level = Now.Level + 1;
                C2.Level = Now.Level + 1;
                C3.Level = Now.Level + 1;
                C4.Level = Now.Level + 1;
                Now.Child1 = C1;
                Now.Child2 = C2;
                Now.Child3 = C3;
                Now.Child4 = C4;
                print("继续分裂:" + nx + " " + ny);
                IsOpen[nx, ny] = true;
                S.Enqueue(C1);
                S.Enqueue(C2);
                S.Enqueue(C3);
                S.Enqueue(C4);
            }
            else
            {
                IsOpen[nx, ny] = false;
            }
        }
        /*
        for(int i=0;i<N;i++)
        {
            print(IsOpen[i, 0] + " " + IsOpen[i, 1] + " " + IsOpen[i, 2] + " " + IsOpen[i, 3] + " " + IsOpen[i, 4] + " " + IsOpen[i, 5]);
        }
        */
    }
    public void BuildHugeTerrain()
    {
        int cnt = 0;
        int cnt1 = 0;
        int cnt2 = 0;
        List<Vector3> vp3 = new List<Vector3>();
        List<Vector2> UV = new List<Vector2>();
        List<int> array = new List<int>();
        Queue<TreeNode> S = new Queue<TreeNode>();
        S.Enqueue(root);
        while (S.Count > 0)
        {
            TreeNode Now = S.Dequeue();
            if (Now.Child1 != null && Now.Child2 != null && Now.Child3 != null && Now.Child4 != null)
            {
                TreeNode c1 = Now.Child1;
                TreeNode c2 = Now.Child2;
                TreeNode c3 = Now.Child3;
                TreeNode c4 = Now.Child4;
                S.Enqueue(c1);
                S.Enqueue(c2);
                S.Enqueue(c3);
                S.Enqueue(c4);
            }
            else
            {
                //我们是根据灰度图的高度图来计算点，需要提前做好UV点坐标，才能够得到每个点的真正高度
                //这个点值得注意一下。、
                int nx = Now.Id / N;
                int ny = Now.Id - (int)(Now.Id / N) * N;
                int cx = (int)Mathf.Pow(2, LOD - Now.Level);
                int cy = (int)Mathf.Pow(2, LOD - Now.Level);
                Vector3 one = new Vector3(nx - cx, 0, ny - cy);
                Vector3 two = new Vector3(nx - cx, 0, ny + cy);
                Vector3 three = new Vector3(nx + cx, 0, ny - cy);
                Vector3 fore = new Vector3(nx + cx, 0, ny + cy);
                float Bx = (float)(1.0f / (float)N);
                float Bz = (float)(1.0f / (float)N);
                UV.Add(new Vector2(one.x * Bx, one.z * Bz));
                UV.Add(new Vector2(two.x * Bx, two.z * Bz));
                UV.Add(new Vector2(three.x * Bx, three.z * Bz));
                UV.Add(new Vector2(fore.x * Bx, fore.z * Bz));

                one.y = GetCellHeight(heightmaps, new Vector2(one.x * Bx, one.z * Bz));
                two.y = GetCellHeight(heightmaps, new Vector2(two.x * Bx, two.z * Bz));
                three.y = GetCellHeight(heightmaps, new Vector2(three.x * Bx, three.z * Bz));
                fore.y = GetCellHeight(heightmaps, new Vector2(fore.x * Bx, fore.z * Bz));


                vp3.Add(one);
                vp3.Add(two);
                vp3.Add(three);
                vp3.Add(fore);

                cnt += 4; cnt1 += 4;
                array.Add(cnt1 - 4);
                array.Add(cnt1 - 3);
                array.Add(cnt1 - 1);
                /*
                array.Add(cnt1 - 1);
                array.Add(cnt1 - 3);
                array.Add(cnt1 - 4);
                */
                array.Add(cnt1 - 4);
                array.Add(cnt1 - 1);
                array.Add(cnt1 - 2);
                /*
                array.Add(cnt1 - 2);
                array.Add(cnt1 - 1);
                array.Add(cnt1 - 4);
                */
                cnt2 += 6;

            }
        }
        print(cnt + "   " + cnt1 + "    " + cnt2);
        Vector2[] uv = new Vector2[cnt];
        Vector3[] vpp = new Vector3[cnt1];
        int[] arrray = new int[cnt2];
        for (int i = 0; i < cnt; i++) uv[i] = UV[i];
        for (int i = 0; i < cnt1; i++) vpp[i] = vp3[i];
        for (int i = 0; i < cnt2; i++) arrray[i] = array[i];
        Mesh mesh = gameObject.GetComponent<MeshFilter>().mesh;
        mesh.vertices = vpp;
        mesh.uv = uv;
        mesh.triangles = arrray;
        gameObject.GetComponent<Renderer>().material = tempmaterial;
        //重置法线
        mesh.RecalculateNormals();
        //重置范围
        mesh.RecalculateBounds();
        /*
        for (int i = 0; i < cnt; i++) 
        {
            print(vpp[i].x + " " + vpp[i].z + "Uv：" + uv[i].x + " " +uv[i].y);
        }
        */
        print(cnt2 / 3);
    }

    public float GetCellHeight(Texture2D map, Vector2 uv)
    {
        // 如果贴图不为空
        if (map != null)
        {
            Color c = GetCellColor(map, uv);
            float gray = c.grayscale;
            float h = H * gray;
            return h;
        }
        else return 0;
    }
    public Color GetCellColor(Texture2D map, Vector2 uv)
    {
        Color color = map.GetPixel(Mathf.FloorToInt(map.width * uv.x), Mathf.FloorToInt(map.height * uv.y));
        return color;
    }

    public float GetDistance(Vector3 A, Vector3 B)
    {
        float Dis;
        //print("A:" + A.x + " " + A.y + " " + A.z);
        //print("B:" + B.x + " " + B.y + " " + B.z);
        Dis = Mathf.Sqrt((A.x - B.x) * (A.x - B.x) + (A.y - B.y) * (A.y - B.y) + (A.z - B.z) * (A.z - B.z));
        //print("Dis:"+Dis);
        return Dis;
    }
}
```









