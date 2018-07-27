```c#

using System.Collections;
using System.Collections.Generic;
using UnityEngine;
//我们的二维网格图左上角第一个位子为（0，0），编码为0；
[RequireComponent(typeof(MeshFilter))]
[RequireComponent(typeof(MeshRenderer))]
[RequireComponent(typeof(MeshCollider))]
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
    public void Start()
    {
        print("yes");
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
        mesh.Clear();
        mesh.vertices = vpp;
        mesh.uv = uv;
        mesh.triangles = arrray;
        //重置法线
        mesh.RecalculateNormals();
        //重置范围
        mesh.RecalculateBounds();
        gameObject.GetComponent<Renderer>().material = tempmaterial;
        gameObject.GetComponent<MeshCollider>().sharedMesh = mesh;
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

using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;

[CustomEditor(typeof(LowerTerrain)),CanEditMultipleObjects]
public class LowerTerrainEditor : Editor {

    Texture2D logo;
    LowerTerrain terrain;
    Vector2 scrollPos;
    /*
    public Texture2D heightmaps;

    public Material tempmaterial;
     */

    [MenuItem("GameObject/3D Object/Huge Terrain")]
    static public void CreatTerrain()
    {
        GameObject gameobject = new GameObject("Huge Terrain");
        gameobject.AddComponent<LowerTerrain>();
        gameobject.AddComponent<MeshFilter>();
        gameobject.AddComponent<MeshCollider>();
        gameobject.AddComponent<MeshRenderer>();
        Selection.activeGameObject = gameobject;
    }
    public override void OnInspectorGUI()
    {
        terrain = (LowerTerrain)target;
        logo = (Texture2D)Resources.Load("NetEase");
        scrollPos = EditorGUILayout.BeginScrollView(scrollPos);
        {

            GUIContent btnTxt = new GUIContent(logo);
            var rt = GUILayoutUtility.GetRect(btnTxt, GUI.skin.label, GUILayout.ExpandWidth(false));

            //设置一个编辑模块
            rt.center = new Vector2(EditorGUIUtility.currentViewWidth / 2, rt.center.y);
            GUI.Button(rt, btnTxt, GUI.skin.label);
            EditorGUI.indentLevel++;
            EditorGUILayout.LabelField("----------在创建地形之前，注意要填写好地形相关信息");
            EditorGUILayout.LabelField("----------注意分层数要根据地形长宽来设定，不要太大");
            terrain.N = EditorGUILayout.IntField("地形长宽：", terrain.N);
            terrain.H = EditorGUILayout.IntField("地形高度：", terrain.H);
            terrain.LOD = EditorGUILayout.IntField("地形分层数：",terrain.LOD);
            terrain.C = EditorGUILayout.Slider("地形分辨率（值越大越精细）：", terrain.C, 0f, 100f);
            terrain.tempmaterial = (Material)EditorGUILayout.ObjectField("地形材质 ：", terrain.tempmaterial, typeof(Material));
            terrain.heightmaps = (Texture2D)EditorGUILayout.ObjectField("地形高度图：", terrain.heightmaps, typeof(Texture2D));
            EditorGUILayout.BeginHorizontal();
            if (GUILayout.Button(new GUIContent("Build Huge Terrain", "Build this Terrain")))
            {
                terrain.Start();
            }
            EditorGUILayout.EndHorizontal();
        }
        EditorGUILayout.EndScrollView();

    }

}

using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class Source : MonoBehaviour
{

    const double eps = 0.0001;

    Mesh mesh;
    //管道横截面积
    private float A = 1;
    //重力加速度
    private float g = 9.8f;
    //管道长度
    private float l = 1;

    //格点间距，按道理来讲不应该设定为1
    private float lx = 1;
    private float ly = 1;

    //增加的水量
    public float Add;

    //源点
    public Vector3 S;

    //高度图b
    private float[,] heightmaps;
    //水高图d
    private float[,] waterheightmaps = new float[600, 600];

    //流量图F 0上，1下，2左 3右 
    private float[,,] F = new float[600, 600, 5];
    private bool[,] IsDown = new bool[600, 600];

    //关于地形的信息
    TerrainData terraindata;
    int width;
    int length;
    int heightResolution;
    int realwidth;
    int reallength;
    int realheight;
    public void AddWater()
    {
        int tx = GetiX(S);
        int tz = GetiZ(S);
        waterheightmaps[tz, tx] += GetiY(Add);
        Add = 0;
        print("YES");
        DrawMesh();
    }
    public void GetHeightMaps()
    {
        print("Yes");
        Terrain terrain = Terrain.activeTerrain;
        terraindata = terrain.terrainData;
        print("Yes");
        width = terraindata.heightmapWidth;
        length = terraindata.heightmapHeight;
        heightmaps = terraindata.GetHeights(0, 0, width, length);
        heightResolution = terraindata.heightmapResolution;
        realwidth = (int)terraindata.size.x;
        realheight = (int)terraindata.size.y;
        reallength = (int)terraindata.size.z;

        print(realwidth + "    " + reallength + "     " + realheight + "     " + heightResolution);
        print(width + "       " + length);
    }
    public void GetPoint(Vector3 pos)
    {
        S = pos;
    }
    public void DoWork(int tim)
    {
        //流量图F 0右，1左，2上 3下 
        int[] fx = { -1, 1, 0, 0 };
        int[] fz = { 0, 0, -1, 1 };
        for (int z = 0; z < tim; z++)
        {
            float dertat = 1f;
            //第一次遍历整个图，来进行水通量的计算。
            for (int i = 0; i < width; i++)
            {
                for (int j = 0; j < length; j++)
                {
                    for (int k = 0; k < 4; k++) F[j, i, k] = 0;
                    if (getY(waterheightmaps[j, i]) <= eps) continue;
                    for (int k = 0; k < 4; k++)
                    {
                        int ni = i + fx[k];
                        int nj = j + fz[k];
                        if (ni >= 0 && ni < width && nj >= 0 && nj < length)
                        {
                            //计算流通量的过程用在真实环境中的地形高度来决定。
                            float CH = getY(heightmaps[j, i] + waterheightmaps[j, i]) - getY(heightmaps[nj, ni] + waterheightmaps[nj, ni]);
                            if (getY(heightmaps[j, i] + waterheightmaps[j, i]) - getY(heightmaps[nj, ni] + waterheightmaps[nj, ni]) <= eps)
                            {
                                F[j, i, k] = 0;
                                continue;
                            }
                            F[j, i, k] = Mathf.Max(0, F[j, i, k] + dertat * A * g * CH / l);
                            if (F[j, i, k] < 0) F[j, i, k] = 0;
                            
                        }
                    }
                    //计算流动系数
                    float K = Mathf.Min(0.9f, getY(waterheightmaps[j, i]) * lx * ly / ((F[j, i, 0] + F[j, i, 1] + F[j, i, 2] + F[j, i, 3]+4f) * dertat));
                    for (int k = 0; k < 4; k++)
                    {
                        F[j, i, k] = F[j, i, k] * K;
                       // print("方向:" + k + "流通值" + F[j, i, k]);
                    }
                }
            }
            //第二次遍历整个图，去计算每个点最终变化的情况
            for (int i = 0; i < width; i++)
            {
                for (int j = 0; j < length; j++)
                {
                    if (i - 1 >= 0 && i - 1 < width && i + 1 >= 0 && i + 1 < width && j - 1 >= 0 && j - 1 < length && j + 1 >= 0 && j + 1 < length)
                    {
                        float Tot = dertat * ((F[j - 1, i, 3] + F[j + 1, i, 2] + F[j, i - 1, 1] + F[j, i + 1, 0]) - (F[j, i, 0] + F[j, i, 1] + F[j, i, 2] + F[j, i, 3]));
                        waterheightmaps[j, i] = waterheightmaps[j, i] + GetiY(Tot / (lx * ly));
                        if (getY(waterheightmaps[j, i]) <= eps) waterheightmaps[j, i] = 0;
                    }
                }
            }
        }
        DrawMesh();
    }
    public void DrawMesh()
    {
        Vector2[] UV = new Vector2[10500000];
        Vector3[] vp3 = new Vector3[10500000];
        int[] array = new int[10500000];
        int cnt, cnt1, cnt2;
        cnt2 = cnt1 = cnt = 0;
        mesh = gameObject.GetComponent<MeshFilter>().mesh;
        mesh.Clear();
        for (int i = 0; i < width; i++)
        {
            for (int j = 0; j < length; j++)
            {
                if (getY(waterheightmaps[j, i]) <= eps) continue;
                
                Vector3 one = new Vector3(getX(i), getY(heightmaps[j, i]) + 0.05f, getZ(j));
                Vector3 two = new Vector3(getX(i), getY(heightmaps[j + 1, i]) + 0.05f, getZ(j + 1));
                Vector3 three = new Vector3(getX(i + 1), getY(heightmaps[j + 1, i +1]) + 0.05f, getZ(j + 1));
                Vector3 fore = new Vector3(getX(i + 1), getY(heightmaps[j, i +1]) + 0.05f, getZ(j));
                
                Vector3 five = new Vector3(getX(i), getY(heightmaps[j, i] + waterheightmaps[j, i]) + 0.05f, getZ(j));
                Vector3 six = new Vector3(getX(i), getY(heightmaps[j + 1, i] + waterheightmaps[j, i]) + 0.05f, getZ(j + 1));
                Vector3 seven = new Vector3(getX(i + 1), getY(heightmaps[j + 1, i + 1] + waterheightmaps[j, i]) + 0.05f, getZ(j + 1));
                Vector3 eight = new Vector3(getX(i + 1), getY(heightmaps[j, i + 1] + waterheightmaps[j, i]) + 0.05f, getZ(j));
                /*
                print("Mesh:" + getY(heightmaps[j, i] + waterheightmaps[j, i]) + 0.05f);

                print("Mesh:" + getY(heightmaps[j + 1, i] + waterheightmaps[j, i]) + 0.05f);

                print("Mesh:" +  getY(heightmaps[j + 1, i - 1] + waterheightmaps[j, i]) + 0.05f);

                print("Mesh:" + getY(heightmaps[j, i - 1] + waterheightmaps[j, i]) + 0.05f);

                */
                UV[cnt++] = new Vector2(one.x, one.y);
                UV[cnt++] = new Vector2(two.x, two.y);
                UV[cnt++] = new Vector2(three.x, three.y);
                UV[cnt++] = new Vector2(fore.x, fore.y);
                
                UV[cnt++] = new Vector2(five.x, five.y);
                UV[cnt++] = new Vector2(six.x, six.y);
                UV[cnt++] = new Vector2(seven.x, seven.y);
                UV[cnt++] = new Vector2(eight.x, eight.y);
                
                vp3[cnt1++] = one;
                vp3[cnt1++] = two;
                vp3[cnt1++] = three;
                vp3[cnt1++] = fore;
                
                vp3[cnt1++] = five;
                vp3[cnt1++] = six;
                vp3[cnt1++] = seven;
                vp3[cnt1++] = eight;
                
                //1 2 3 4
                array[cnt2++] = cnt1 - 8;
                array[cnt2++] = cnt1 - 7;
                array[cnt2++] = cnt1 - 6;

                array[cnt2++] = cnt1 - 6;
                array[cnt2++] = cnt1 - 7;
                array[cnt2++] = cnt1 - 8;
                
                array[cnt2++] = cnt1 - 8;
                array[cnt2++] = cnt1 - 6;
                array[cnt2++] = cnt1 - 5;

                array[cnt2++] = cnt1 - 5;
                array[cnt2++] = cnt1 - 6;
                array[cnt2++] = cnt1 - 8;
                
                //5 6 7 8
                array[cnt2++] = cnt1 - 4;
                array[cnt2++] = cnt1 - 3;
                array[cnt2++] = cnt1 - 2;

                array[cnt2++] = cnt1 - 2;
                array[cnt2++] = cnt1 - 3;
                array[cnt2++] = cnt1 - 4;
                
                array[cnt2++] = cnt1 - 4;
                array[cnt2++] = cnt1 - 2;
                array[cnt2++] = cnt1 - 1;

                array[cnt2++] = cnt1 - 1;
                array[cnt2++] = cnt1 - 2;
                array[cnt2++] = cnt1 - 4;

                //6 5 2 1

                array[cnt2++] = cnt1 - 3;
                array[cnt2++] = cnt1 - 4;
                array[cnt2++] = cnt1 - 7;

                array[cnt2++] = cnt1 - 7;
                array[cnt2++] = cnt1 - 4;
                array[cnt2++] = cnt1 - 3;

                array[cnt2++] = cnt1 - 8;
                array[cnt2++] = cnt1 - 7;
                array[cnt2++] = cnt1 - 3;

                //7 6 3 2

                array[cnt2++] = cnt1 - 2;
                array[cnt2++] = cnt1 - 3;
                array[cnt2++] = cnt1 - 6;

                array[cnt2++] = cnt1 - 6;
                array[cnt2++] = cnt1 - 3;
                array[cnt2++] = cnt1 - 2;

                array[cnt2++] = cnt1 - 2;
                array[cnt2++] = cnt1 - 6;
                array[cnt2++] = cnt1 - 7;

                array[cnt2++] = cnt1 - 7;
                array[cnt2++] = cnt1 - 6;
                array[cnt2++] = cnt1 - 2;

                //8 7 4 3
                array[cnt2++] = cnt1 - 1;
                array[cnt2++] = cnt1 - 2;
                array[cnt2++] = cnt1 - 5;

                array[cnt2++] = cnt1 - 5;
                array[cnt2++] = cnt1 - 2;
                array[cnt2++] = cnt1 - 1;

                array[cnt2++] = cnt1 - 1;
                array[cnt2++] = cnt1 - 5;
                array[cnt2++] = cnt1 - 6;

                array[cnt2++] = cnt1 - 6;
                array[cnt2++] = cnt1 - 5;
                array[cnt2++] = cnt1 - 1;

                //8 5 4 2

                array[cnt2++] = cnt1 - 1;
                array[cnt2++] = cnt1 - 4;
                array[cnt2++] = cnt1 - 5;

                array[cnt2++] = cnt1 - 5;
                array[cnt2++] = cnt1 - 4;
                array[cnt2++] = cnt1 - 1;

                array[cnt2++] = cnt1 - 1;
                array[cnt2++] = cnt1 - 5;
                array[cnt2++] = cnt1 - 7;

                array[cnt2++] = cnt1 - 7;
                array[cnt2++] = cnt1 - 5;
                array[cnt2++] = cnt1 - 1;



            }
        }
        print(cnt + "     " + cnt1 + "     " + cnt2);
        Vector2[] uv = new Vector2[cnt];
        Vector3[] vp = new Vector3[cnt1];
        int[] arrray = new int[cnt2];
        for (int i = 0; i < cnt; i++) uv[i] = UV[i];
        for (int i = 0; i < cnt1; i++) vp[i] = vp3[i];
        for (int i = 0; i < cnt2; i++) arrray[i] = array[i];
        mesh.vertices = vp;
        mesh.uv = uv;
        mesh.triangles = arrray;
    }

    int GetiX(Vector3 P)
    {
        float Bilv1 = (float)(heightResolution) / (float)(realwidth);
        int tx = (int)(P.x * Bilv1);
        return tx;
    }
    int GetiZ(Vector3 P)
    {
        float Bilv2 = (float)(heightResolution) / (float)(reallength);
        int tz = (int)(P.z * Bilv2);
        return tz;
    }
    float GetiY(float y)
    {
        float Bilv3 = 1.0f / (float)(realheight);
        float ty = y * Bilv3;
        return ty;
    }
    float getX(float x)
    {
        return (float)(x) * (float)(realwidth) / (float)(heightResolution);
    }
    float getY(float y)
    {
        return (float)(y) * (float)(realheight);
    }
    float getZ(float z)
    {
        return (float)(z) * (float)(reallength) / (float)(heightResolution);
    }
}
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;
[CustomEditor(typeof(Source)), CanEditMultipleObjects]
public class SourceEditor : Editor {

    private bool IsHave=false;
    Texture2D logo;
    Source source;
    Vector2 scrollPos;

    [MenuItem("GameObject/3D Object/Source")]
    static public void CreatSource()
    {
        GameObject gameobject = new GameObject("Source");
        gameobject.AddComponent<MeshFilter>();
        gameobject.AddComponent<MeshRenderer>();
        gameobject.AddComponent<Source>();
        Selection.activeGameObject = gameobject;
    }
    public override void OnInspectorGUI()
    {
        //进行数据交互之前必须调用target
        source = (Source)target;
        logo = (Texture2D)Resources.Load("NetEase");
        scrollPos = EditorGUILayout.BeginScrollView(scrollPos);
        {
            
            GUIContent btnTxt = new GUIContent(logo);
            var rt = GUILayoutUtility.GetRect(btnTxt, GUI.skin.label, GUILayout.ExpandWidth(false));

            //设置一个编辑模块
            rt.center = new Vector2(EditorGUIUtility.currentViewWidth / 2, rt.center.y);

            GUI.Button(rt, btnTxt, GUI.skin.label);
            
            EditorGUILayout.LabelField("选取水源源点 --- 在地形上单机鼠标左键");
            EditorGUI.indentLevel++;

            EditorGUILayout.BeginHorizontal();
            EditorGUILayout.LabelField("Point:");
            source.S = EditorGUILayout.Vector3Field("", source.S);
            EditorGUILayout.EndHorizontal();

            EditorGUILayout.BeginHorizontal();
            source.Add= EditorGUILayout.FloatField("在源点处增加水量值", source.Add);
            if (GUILayout.Button(new GUIContent("增加水量", "")))
            {
                source.GetHeightMaps();
                source.AddWater();
            }
            EditorGUILayout.EndHorizontal();

            EditorGUILayout.BeginHorizontal();
            if (GUILayout.Button(new GUIContent("进行1单位时间", "")))
            {
                source.DoWork(1);
            }
            EditorGUILayout.EndHorizontal();
            EditorGUILayout.BeginHorizontal();
            if (GUILayout.Button(new GUIContent("进行100单位时间", "")))
            {
                source.DoWork(100);
            }
            EditorGUILayout.EndHorizontal();




        }
        EditorGUILayout.EndScrollView();
        
        

    }
    protected virtual void OnSceneGUI()
    {
        int controlId = GUIUtility.GetControlID(FocusType.Passive);
        if(source!=null)
        {
            if(IsHave==true)
            {
                EditorGUI.BeginChangeCheck();
                Handles.color = new Color32(147, 225, 58, 255);
                Vector3 handlePos = (Vector3)source.S + source.transform.position;
                if (Tools.current == Tool.Move)
                {
                    float size = 0.6f;

                    Handles.color = Handles.xAxisColor;
                    Vector3 pos = Handles.Slider((Vector3)source.S + source.transform.position, Vector3.right, HandleUtility.GetHandleSize(handlePos) * size, Handles.ArrowHandleCap, 0.01f) - source.transform.position;

                    pos = Handles.Slider((Vector3)pos + source.transform.position, Vector3.up, HandleUtility.GetHandleSize(handlePos) * size, Handles.ArrowHandleCap, 0.01f) - source.transform.position;

                    pos = Handles.Slider((Vector3)pos + source.transform.position, Vector3.forward, HandleUtility.GetHandleSize(handlePos) * size, Handles.ArrowHandleCap, 0.01f) - source.transform.position;

                    source.S = pos;
                }
                if (Tools.current == Tool.Scale)
                {
                    Handles.color = Color.red;
                    Vector3 pos = source.S;
                    source.S = pos;
                }
                if (EditorGUI.EndChangeCheck())
                {
                    Undo.RecordObject(source, "Change Position");
                }
            }
        }
        //监听鼠标和crtl键是否同时按下，如果是，得到shaded中的那个点、
        if (Event.current.type == EventType.MouseDown && Event.current.button == 0 && Event.current.control)
        {
            if(IsHave==false)
            {
                Vector3 screenPosition = Event.current.mousePosition;
                screenPosition.y = Camera.current.pixelHeight - screenPosition.y;
                Ray ray = Camera.current.ScreenPointToRay(screenPosition);
                RaycastHit hit;

                if (Physics.Raycast(ray, out hit))
                {
                    Undo.RecordObject(source, "Add point");

                    Vector3 position = hit.point - source.transform.position;
                    source.GetPoint(position);

                    GUIUtility.hotControl = controlId;
                    Event.current.Use();
                    HandleUtility.Repaint();
                }
                IsHave = true;
            }
            
        }
        if (Event.current.type == EventType.MouseUp && Event.current.button == 0 && Event.current.control)
        {
            GUIUtility.hotControl = 0;
        }
    }
}

```
