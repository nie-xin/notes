OSG 笔记
===

##The scene - la scène

A top-down and tree-like structure graphical scene.

Nodes are the aspects of scene

* Leaf node = geometry to be rendered
* Non-leaf node = hierarchy ans scene manipulations
 
###Important classes

osg::Node

* nodes in a scene graph
* subclasses: osg::Geode, osg::Group
	
osg::Drawable

* Not a node so it must be attached to a node
* Actual rendered geometry
* subclasses: osg::Geometry, osg::ShapeDrawable

osgViewer::Viewer - controls the view of scene

###Set up a scene
1. define the root node(eg.osg::Group)
2. define elements 
3. make drawable shape
4. attach drawable shape to graph node 
5. add graph node to root node
	

###StateSet
StateSet define the OpengGL state for a node
* texture
* culling (hidden surface = 剔除)
* lighting
	
###View
* set the scene graph
* set the camera
* set up the window
	
	
##Some useful patterns
*visitor - traversal and rendering the scene graph
*observer - notifications for goups of nodes
*decorator - dynamically change behaviors 
	
##Alternatives
OGRE (Object-Oriented Graphics Rendering Engine)
http://www.ogre3d.org/

---

##场景渲染方式

1. 更新：更新遍历可以修改场景图形，以实现动态场景。更新操作由回调函数完成。
2. 拣选：拣选遍历选择渲染在是口内的节点。
3. 绘制：绘制遍历根据拣选遍历的过程生成几何体并实现渲染


