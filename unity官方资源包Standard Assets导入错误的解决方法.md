---
title: unity官方资源包Standard Assets导入错误的解决方法
date: 2020-07-02 12:06:24
tags: Unity
keywords: Unity
categories: 实操
---

# unity官方资源包Standard Assets导入错误的解决方法

### unity官方资源包Standard Assets导入错误

使用unity2020时，导入unity官方资源包Standard Assets出错了
然后试了试2019、2018的版本，居然都不行
unity2017倒没问题
我惊了，官方都没发现这个bug吗
仔细看了看Console，知道哪里出问题了

**\Assets\Standard Assets\Utility\ForcedReset.cs**
和
**\Assets\Standard Assets\Utility\SimpleActivatorMenu.cs**
出错了，大概是因为新版的unity没有GUITexture这类库了吧

把**ForcedReset.cs**的**GUITexture**修改为**UnityEngine.UI.Image**
**SimpleActivatorMenu.cs**的**GUITexture**修改为**UnityEngine.UI.Text**

再运行时，就没有问题了

也就是说，把**ForcedReset.cs**改为

~~~c#
using System;
using UnityEngine;
using UnityEngine.SceneManagement;
using UnityStandardAssets.CrossPlatformInput;

[RequireComponent(typeof (UnityEngine.UI.Image))]
public class ForcedReset : MonoBehaviour
{
    private void Update()
    {
        // if we have forced a reset ...
        if (CrossPlatformInputManager.GetButtonDown("ResetObject"))
        {
            //... reload the scene
            SceneManager.LoadScene(SceneManager.GetSceneAt(0).name);
        }
    }
}

~~~

把**SimpleActivatorMenu.cs**改为

~~~c#
using System;
using UnityEngine;

namespace UnityStandardAssets.Utility
{
    public class SimpleActivatorMenu : MonoBehaviour
    {
        // An incredibly simple menu which, when given references
        // to gameobjects in the scene
        public UnityEngine.UI.Text camSwitchButton;
        public GameObject[] objects;


        private int m_CurrentActiveObject;


        private void OnEnable()
        {
            // active object starts from first in array
            m_CurrentActiveObject = 0;
            camSwitchButton.text = objects[m_CurrentActiveObject].name;
        }


        public void NextCamera()
        {
            int nextactiveobject = m_CurrentActiveObject + 1 >= objects.Length ? 0 : m_CurrentActiveObject + 1;

            for (int i = 0; i < objects.Length; i++)
            {
                objects[i].SetActive(i == nextactiveobject);
            }

            m_CurrentActiveObject = nextactiveobject;
            camSwitchButton.text = objects[m_CurrentActiveObject].name;
        }
    }
}


~~~

大概是因为这个bug太简单了以至于官方没当回事吧hhhh
 不过对于新手来说确实是个麻烦

希望可以帮到你