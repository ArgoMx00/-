一、用户使用说明

①右键在有地形的场景中，创建RiverTool，得到一个河流编辑器。

②然后根据用户Inspector界面提示信息，将需要填写的地形信息进行填写。用户界面有一些简单操作的描述

![](https://i.loli.net/2018/06/24/5b2f4014b6279.png)

![](https://i.loli.net/2018/06/24/5b2f43573d0ae.png)

③接下来我们在编辑界面，按住Crtl+鼠标左键点选路径点，并且设置控制柄

④因为符合逻辑作业，所以我们需要先点击创建河床再点击创建河流才行，否则不能正常运行，即第一步必须点击创建河床。

⑤有功能：建立河床、删除河床、建立河流、删除河流、设置基础shader。

编辑模式下的实际效果图：

![](https://i.loli.net/2018/06/24/5b2f43f8be6c8.png)


缺点记录：

对于当前功能来讲，河流不能动的问题应该尽快解决。

之后增加河流移动的功能的时候，一定要安排好方向，写好shader之类的东西，进行学习一哈。


二、源码保存:

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;
[CustomEditor(typeof(Build_River_Tool)), CanEditMultipleObjects]
public class Build_River_ToolEditor : Editor
{

    int flag = 1;
    Build_River_Tool rivertool;
    bool showPositions = false;
    Vector2 scrollPos;
    Texture2D logo;
    [MenuItem("GameObject/3D Object/Creat RiverTool")]
    static public void CreateRierTool()
    {
        GameObject gameobject = new GameObject("RiverTool");
        gameobject.AddComponent<Build_River_Tool>();
        gameobject.AddComponent<MeshFilter>();
        gameobject.AddComponent<MeshRenderer>();
        Selection.activeGameObject = gameobject;
    }
    public override void OnInspectorGUI()
    {
        //进行数据交互前必须调到他的target；
        rivertool = (Build_River_Tool)target;
        logo = (Texture2D)Resources.Load("NetEase");

        scrollPos = EditorGUILayout.BeginScrollView(scrollPos);
        {
            GUIContent btnTxt = new GUIContent(logo);

            var rt = GUILayoutUtility.GetRect(btnTxt, GUI.skin.label, GUILayout.ExpandWidth(false));

            //设置一个编辑模块
            rt.center = new Vector2(EditorGUIUtility.currentViewWidth / 2, rt.center.y);

            GUI.Button(rt, btnTxt, GUI.skin.label);
            EditorGUILayout.LabelField("注意-----请先添加地形的信息,否则无法进行河流建立工作");
            rivertool.realWidth = EditorGUILayout.IntField("地形宽度", rivertool.realWidth);
            rivertool.realLength = EditorGUILayout.IntField("地形长度", rivertool.realLength);
            rivertool.realHeight = EditorGUILayout.IntField("地形高度", rivertool.realHeight);
            rivertool.heightResolution = EditorGUILayout.IntField("地形分辨率", rivertool.heightResolution);
            rivertool.terrainData = (TerrainData)EditorGUILayout.ObjectField("地形来源", rivertool.terrainData, typeof(TerrainData));
            EditorGUILayout.LabelField("调试河流参数");
            float aa = 0f; float bb = 10f;
            rivertool.RiverDepth = EditorGUILayout.Slider("河流深度", rivertool.RiverDepth, 0f, 10f);
            rivertool.Slope_Val = EditorGUILayout.Slider("河流坡度", rivertool.Slope_Val, 0f, 5f);
            rivertool.Height_Del = EditorGUILayout.Slider("河流每次下降的高度", rivertool.Height_Del, 0f, 0.5f);
            rivertool.AAwidth = EditorGUILayout.IntSlider("处理点周围宽度（河流宽度）", rivertool.AAwidth, 0, 15);
            rivertool.AAlength = EditorGUILayout.IntSlider("处理点周围长度（点间距）", rivertool.AAlength, 0, 15);
            EditorGUI.indentLevel++;

            //增加一个点的Folder
            EditorGUILayout.LabelField("注意-----每次选定完路径点之后");
            EditorGUILayout.LabelField("注意-----请先点击一次创建河床");
            EditorGUILayout.LabelField("注意-----再进行其他操作");

            EditorGUILayout.BeginHorizontal();
            showPositions = EditorGUILayout.Foldout(showPositions, "Points");
            EditorGUILayout.EndHorizontal();
            if (showPositions)
            {
                for (int i = 0; i < rivertool.cnt; i++)
                {
                    EditorGUILayout.BeginHorizontal();
                    rivertool.m_vp3River[i] = EditorGUILayout.Vector3Field("", rivertool.m_vp3River[i]);
                    if (GUILayout.Button(new GUIContent("R", "Remove this Point")))
                    {
                        Undo.RecordObject(rivertool, "Remove point");
                        rivertool.RemovePos(i);
                    }
                    EditorGUILayout.EndHorizontal();
                }
            }
            EditorGUILayout.BeginHorizontal();
            if (GUILayout.Button(new GUIContent("Build RiverBed", "Build this RiverBed")))
            {
                Undo.RecordObject(rivertool, "BuildRiverBed");
                rivertool.BuildRiverBed();
            }
            if (GUILayout.Button(new GUIContent("Close RiverBed", "Close This RiverBed")))
            {
                Undo.RecordObject(rivertool, "CloseRiverBed");
                rivertool.CloseRiverBed();
            }
            EditorGUILayout.EndHorizontal();
            EditorGUILayout.BeginHorizontal();
            if (GUILayout.Button(new GUIContent("Build River", "Build this River")))
            {
                Undo.RecordObject(rivertool, "BuildRiver");
                rivertool.BuildR();
            }
            if (GUILayout.Button(new GUIContent("Close River", "Close this River")))
            {
                Undo.RecordObject(rivertool, "CloseRiver");
                rivertool.CloseR();
            }
            EditorGUILayout.EndHorizontal();
            EditorGUILayout.BeginHorizontal();
            if (GUILayout.Button("Set basic river material"))
            {
                try
                {

                    string materialName = "WaterBasicDaytime";
                    if (PlayerSettings.colorSpace == ColorSpace.Linear)
                        materialName = "WaterBasicDaytime";

                    Material riverMat = (Material)Resources.Load(materialName);

                    if (riverMat != null)
                    {
                        rivertool.GetComponent<MeshRenderer>().sharedMaterial = riverMat;
                    }


                }
                catch
                {


                }

            }
        }
        EditorGUILayout.EndScrollView();
    
    }
    protected virtual void OnSceneGUI()
    {
        //对n个点，设立n个控制柄
        int controlId = GUIUtility.GetControlID(FocusType.Passive);
        
        for(int i=0;i<rivertool.cnt;i++)
        {
            EditorGUI.BeginChangeCheck();
            Handles.color = new Color32(147, 225, 58, 255);
            Vector3 handlePos = (Vector3)rivertool.m_vp3River[i] + rivertool.transform.position;
            if(Tools.current==Tool.Move)
            {
                float size = 0.6f;

                Handles.color = Handles.xAxisColor;
                Vector3 pos = Handles.Slider((Vector3)rivertool.m_vp3River[i] + rivertool.transform.position, Vector3.right, HandleUtility.GetHandleSize(handlePos) * size, Handles.ArrowHandleCap, 0.01f) - rivertool.transform.position;

                pos = Handles.Slider((Vector3)pos + rivertool.transform.position, Vector3.up, HandleUtility.GetHandleSize(handlePos) * size, Handles.ArrowHandleCap, 0.01f) - rivertool.transform.position;

                pos = Handles.Slider((Vector3)pos + rivertool.transform.position, Vector3.forward, HandleUtility.GetHandleSize(handlePos) * size, Handles.ArrowHandleCap, 0.01f) - rivertool.transform.position;

                rivertool.m_vp3River[i] = pos;
            }
            if(Tools.current==Tool.Scale)
            {
                Handles.color = Color.red;
                Vector3 pos = rivertool.m_vp3River[i];
                rivertool.m_vp3River[i] = pos;
            }
            
            if (EditorGUI.EndChangeCheck())
            {
                Undo.RecordObject(rivertool, "Change Position");
                rivertool.BuildR();
            }
        }
        //监听鼠标和crtl键是否同时按下，如果是，得到shaded中的那个点、
        if (Event.current.type == EventType.MouseDown && Event.current.button == 0 && Event.current.control)
        {
            Vector3 screenPosition = Event.current.mousePosition;
            screenPosition.y = Camera.current.pixelHeight - screenPosition.y;
            Ray ray = Camera.current.ScreenPointToRay(screenPosition);
            RaycastHit hit;

            if (Physics.Raycast(ray, out hit))
            {
                Undo.RecordObject(rivertool, "Add point");

                Vector4 position = hit.point - rivertool.transform.position;
                position.w = 1;
                rivertool.GetPoint(position);

                GUIUtility.hotControl = controlId;
                Event.current.Use();
                HandleUtility.Repaint();
            }
        }
        if (Event.current.type == EventType.MouseUp && Event.current.button == 0 && Event.current.control)
        {
            GUIUtility.hotControl = 0;

        }
        
    }
}

using System.Collections;
using System.Collections.Generic;
using UnityEngine;
[RequireComponent(typeof(MeshFilter))]
[RequireComponent(typeof(MeshRenderer))]
public class Build_River_Tool : MonoBehaviour
{
    /********************************************************************/
    //注意重新将脚本拖入GameObject的时候，将初始化的东西都要初始化一下。
    //首先我们需要采集地形信息，所以我们首先创建出来一个terrainData
    //用来存储需要采集的地形的信息。
    //TerrainData类中元素：http://www.manew.com/youxizz/2763.html
    public TerrainData terrainData;
    public GameObject Water;
    //当前高度图
    public float[,] heightmaps;
    //还原高度图（运行时候初始高度图）
    public float[,] returnheightmaps;
    //水的高度图
    public float[,] waterheightmaps;

    //得到在代码中的长和宽。
    public int width;
    public int length;

    //鼠标点击的点
    Vector3 P;
    //实际地形图中的信息：
    //地形宽度、地形长度、地形高度、地形分辨率。
    public int realWidth;
    public int realLength;
    public int realHeight;
    public int heightResolution;


    //用于保存收集点的信息
    public int cnt;
    public Vector3[] m_vp3River = new Vector3[10000];
    public Vector3[] Tm_vp3River = new Vector3[10000];
    public Vector3[] returnm_vp3 = new Vector3[10000];
    //河流的深度
    public float RiverDepth;
    //预设坡度、如果大于坡度降低的高度值。
    public float Slope_Val;
    public float Height_Del;
    //凹河床周围点的范围
    public int AAwidth;
    public int AAlength;

    //判断是否是右键点击的功能
    public int IsRight;
    /********************************************************************/





    /********************************************************************/
    public void BuildRiverBed()
    {
        for (int i = 0; i < cnt; i++)
        {
            returnm_vp3[i] = m_vp3River[i];
        }
        GetHeightmaps();
        Psort();
        FirstUpdateRiver();
        SecondUpdateRiver();
    }
    public void CloseRiverBed()
    {
        heightmaps = returnheightmaps;
        terrainData.SetHeights(0,0,heightmaps);
        for (int i = 0; i < cnt; i++)
        {
            m_vp3River[i]=returnm_vp3[i];
        }
    }
    //得到当前地形的高度图。
    public void GetHeightmaps()
    {
        width = terrainData.heightmapWidth;
        length = terrainData.heightmapHeight;
        heightmaps = terrainData.GetHeights(0, 0, width, length);
        returnheightmaps = terrainData.GetHeights(0, 0, width, length);
        Debug.Log("得到初始地形");
        //print(width + "---" + height);
    }
    //得到鼠标点选的点。
    public void GetPoint(Vector4 P)
    {
        print(P.x + "  " + P.y + "   " + P.z + cnt);
        m_vp3River[cnt].x = P.x;
        m_vp3River[cnt].y = P.y;
        m_vp3River[cnt].z = P.z;
        cnt++;
    }
    //删除点并且还原地形
    public void RemovePos(int p)
    {
        Vector3 pos = m_vp3River[p];
        for (int i = 0; i < cnt; i++) Tm_vp3River[i] = m_vp3River[i];
        for (int i = 0; i < p; i++) m_vp3River[i] = Tm_vp3River[i];
        for (int i = p; i < cnt - 1; i++) m_vp3River[i] = Tm_vp3River[i + 1];
        cnt--;
    }
    /********************************************************************/






    /********************************************************************/
    //初步处理出河床
    //首先将所有采集到的点按照高度排序。
    void Psort()
    {
        //我们应该先将收集到的所有点按照高度排序。考虑到当前点的个数不多，我们先按照冒泡排序去处理、
        //将数组高度处理为递减数组（地势高的放在数组前边）。
        for (int i = 0; i < cnt; i++)
        {
            for (int j = i; j < cnt; j++)
            {
                if (j + 1 < cnt && j < cnt)
                {
                    if (m_vp3River[j].y < m_vp3River[j + 1].y)
                    {
                        Vector3 Temp;
                        Temp = m_vp3River[j];
                        m_vp3River[j] = m_vp3River[j + 1];
                        m_vp3River[j + 1] = Temp;
                    }

                }
            }
        }
        int tmpcnt = 0;
        //去重点，防止影响排序等做法的时间复杂度。
        for (int i = 0; i < cnt; i++)
        {
            if (i == 0)
            {
                Tm_vp3River[tmpcnt++] = m_vp3River[i];
            }
            else
            {
                if (m_vp3River[i] == m_vp3River[i - 1])
                {
                    continue;
                }
                else
                {
                    Tm_vp3River[tmpcnt++] = m_vp3River[i];
                }
            }
        }
        for (int i = 0; i < tmpcnt; i++)
        {
            m_vp3River[i] = Tm_vp3River[i];
        }
        cnt = tmpcnt;
        print("所有点选点的信息：");
        for (int i = 0; i < cnt; i++)
        {
            print(m_vp3River[i].x + " - " + m_vp3River[i].y + " - " + m_vp3River[i].z);
        }
    }

    //初步保证河流高度不可逆
    void FirstUpdateRiver()
    {
        for (int i = 0; i <cnt; i++)
        {
            int tx = GetiX(m_vp3River[i]);
            int tz = GetiZ(m_vp3River[i]);
            m_vp3River[i].y -= RiverDepth;
            if (i > 0 && m_vp3River[i].y >= m_vp3River[i - 1].y)
            {
                m_vp3River[i].y = m_vp3River[i - 1].y - 0.0001f * realHeight;
            }
            float ty = GetiY(m_vp3River[i]);
            heightmaps[tz, tx] = ty;
        }
        terrainData.SetHeights(0, 0, heightmaps);
    }
    /********************************************************************/




    /********************************************************************/
    void SecondUpdateRiver()
    {
        for (int i = 0; i < cnt; i++)
        {
            int tx = GetiX(m_vp3River[i]);
            int tz = GetiZ(m_vp3River[i]);
            int flag;
            int ccnt = 0;
            while (true)
            {
                ccnt++;
                flag = 0;
                for (int iX = tx - AAwidth; iX <= tx + AAwidth; iX++)
                {
                    for (int iZ = tz - AAlength; iZ <= tz + AAlength; iZ++)
                    {
                        if (iX >= 0 && iX < width && iZ >= 0 && iZ < length)
                        {
                            if (iX == tx && iZ == tz) continue;
                            float Fslope;
                            //两点之间坡度的计算公式：(两点之间高度差)/(两点之间水平距离)
                            //计算两点之间坡度，如果坡度大于预设坡度，那么我们修改这个点的高度。
                            float x1 = (float)((float)(iX) * (float)realWidth) / (float)(heightResolution);
                            float x2 = (float)((float)(tx) * (float)realWidth) / (float)(heightResolution);
                            float z1 = (float)((float)(iZ) * (float)realLength) / (float)(heightResolution);
                            float z2 = (float)((float)(tz) * (float)realLength) / (float)(heightResolution);

                            float Dis = (float)(System.Math.Sqrt((x1 - x2) * (x1 - x2) + (z1 - z2) * (z1 - z2)));
                            Fslope = (heightmaps[iZ, iX] - heightmaps[tz, tx]) * realHeight / Dis;
                            if (Fslope > Slope_Val)
                            {
                                if (heightmaps[iZ, iX] >= Height_Del)
                                {
                                    heightmaps[iZ, iX] -= Height_Del;
                                    flag = 1;
                                }
                            }
                        }
                    }
                }
                if (flag == 0 || ccnt >= 1000000) break;
            }
        }
        terrainData.SetHeights(0, 0, heightmaps);
    }
    public void CloseR()
    {
        Mesh mesh;
        mesh = gameObject.GetComponent<MeshFilter>().mesh;
        mesh.Clear();
    }
    public void BuildR()
    {
        int cnt2 = 0;
        Vector3[] V = new Vector3[150000];
        int[] fx = new int[4];
        int[] fz = new int[4];
        fx[0] = 0; fx[1] = 1; fx[2] = 1; fx[3] = 0;
        fz[0] = 0; fz[1] = 0; fz[2] = 1; fz[3] = 1;
        for (int l = 0; l < cnt; l++)
        {
            int tx = GetiX(m_vp3River[l]);
            int tz = GetiZ(m_vp3River[l]);
            for (int i = tx - AAwidth; i <= tx + AAwidth; i++)
            {
                for (int j = tz - AAlength; j <= tz + AAlength; j++)
                {
                    if (i == tx - AAwidth || i == tx + AAwidth || j == tz - AAlength || j == tz + AAlength) continue;
                    Vector3[] D = new Vector3[4];
                    int flags = 1;
                    for (int k = 0; k < 4; k++)
                    {
                        int iX = i + fx[k];
                        int iZ = j + fz[k];
                        if (iX >= 0 && iX < width && iZ >= 0 && iZ < length)
                        {
                            float x = (float)(iX) * (float)(realWidth) / (float)(heightResolution);
                            float y = heightmaps[iZ, iX] * realHeight;
                            float z = (float)(iZ) * (float)(realLength) / (float)(heightResolution);
                            D[k] = new Vector3(x, y, z);
                        }
                        else flags = 0;
                    }
                    if (flags == 1)
                    {
                        for (int k = 0; k < 4; k++)
                        {
                            V[cnt2++] = D[k];
                        }
                    }
                }
            }
        }
        BuildV(V, cnt2);
    }
    void BuildV(Vector3[] tempV,int cnt)
    {
        Vector3[] vp3 = new Vector3[1050000];
        int cnt1 = 0;
        int cnt2 = 0;
        int[] array = new int[1050000];
        Vector3[] vv = new Vector3[1050000];
        for (int i = 0; i < cnt; i++)
        {
            vv[i] = tempV[i];
        }
        for (int i = 0; i < cnt; i += 4)
        {
            //得到矩形的八个点
            Vector3[] vp = new Vector3[8];
            for (int j = i; j < i + 4; j++)
            {
                vp[j - i] = tempV[j];
            }
            for (int j = i; j < i + 4; j++)
            {
                vp[j - i + 4] = tempV[j];
                vp[j - i + 4].y += 0.3f;
            }
            for (int j = 0; j < 8; j++)
            {
                vp3[cnt1++] = vp[j];
            }
        }
        for (int i = 0; i < cnt1; i += 8)
        {
            //1 2 3 4
            array[cnt2++] = i;
            array[cnt2++] = i + 1;
            array[cnt2++] = i + 2;
            array[cnt2++] = i;
            array[cnt2++] = i + 2;
            array[cnt2++] = i + 3;
            array[cnt2++] = i + 2;
            array[cnt2++] = i + 1;
            array[cnt2++] = i;
            array[cnt2++] = i + 3;
            array[cnt2++] = i + 2;
            array[cnt2++] = i;
            //5 6 7 8
            array[cnt2++] = i + 4;
            array[cnt2++] = i + 5;
            array[cnt2++] = i + 6;
            array[cnt2++] = i + 4;
            array[cnt2++] = i + 6;
            array[cnt2++] = i + 7;
            array[cnt2++] = i + 6;
            array[cnt2++] = i + 5;
            array[cnt2++] = i + 4;
            array[cnt2++] = i + 7;
            array[cnt2++] = i + 6;
            array[cnt2++] = i + 4;
            //5 6 1 2
            array[cnt2++] = i + 4;
            array[cnt2++] = i + 5;
            array[cnt2++] = i + 1;
            array[cnt2++] = i + 4;
            array[cnt2++] = i + 1;
            array[cnt2++] = i;
            array[cnt2++] = i + 1;
            array[cnt2++] = i + 5;
            array[cnt2++] = i + 4;
            array[cnt2++] = i;
            array[cnt2++] = i + 1;
            array[cnt2++] = i + 4;
            //5 8 1 4
            array[cnt2++] = i + 4;
            array[cnt2++] = i + 7;
            array[cnt2++] = i + 3;
            array[cnt2++] = i + 4;
            array[cnt2++] = i + 3;
            array[cnt2++] = i;
            array[cnt2++] = i + 3;
            array[cnt2++] = i + 7;
            array[cnt2++] = i + 4;
            array[cnt2++] = i;
            array[cnt2++] = i + 3;
            array[cnt2++] = i + 4;
            //7 6 3 2
            array[cnt2++] = i + 6;
            array[cnt2++] = i + 5;
            array[cnt2++] = i + 1;
            array[cnt2++] = i + 6;
            array[cnt2++] = i + 1;
            array[cnt2++] = i + 2;
            array[cnt2++] = i + 1;
            array[cnt2++] = i + 5;
            array[cnt2++] = i + 6;
            array[cnt2++] = i + 2;
            array[cnt2++] = i + 1;
            array[cnt2++] = i + 6;
            //8 7 4 3
            array[cnt2++] = i + 7;
            array[cnt2++] = i + 6;
            array[cnt2++] = i + 2;
            array[cnt2++] = i + 7;
            array[cnt2++] = i + 2;
            array[cnt2++] = i + 3;
            array[cnt2++] = i + 2;
            array[cnt2++] = i + 6;
            array[cnt2++] = i + 7;
            array[cnt2++] = i + 3;
            array[cnt2++] = i + 2;
            array[cnt2++] = i + 7;
        }
        Mesh mesh = gameObject.GetComponent<MeshFilter>().mesh;
        mesh.Clear();
        Vector3[] vpp3 = new Vector3[cnt1];
        int[] arrray = new int[cnt2];
        for (int i = 0; i < cnt1; i++) vpp3[i] = vp3[i];
        for (int i = 0; i < cnt2; i++) arrray[i] = array[i];
        print(cnt1 + "--" + cnt2);
        mesh.vertices = vpp3;
        mesh.triangles = arrray;
    }
    /********************************************************************/






    /********************************************************************/
    //将地形图的点换算为代码中的（x，y，z）；
    int GetiX(Vector3 P)
    {
        float Bilv1 = (float)(heightResolution) / (float)(realWidth);
        int tx = (int)(P.x * Bilv1);
        return tx;
    }
    int GetiZ(Vector3 P)
    {
        float Bilv2 = (float)(heightResolution) / (float)(realLength);
        int tz = (int)(P.z * Bilv2);
        return tz;
    }
    float GetiY(Vector3 P)
    {
        float Bilv3 = 1.0f / (float)(realHeight);
        float ty = P.y * Bilv3;
        return ty;
    }
    /********************************************************************/
}

```
