# Activity和Fragment 生命周期 #

**启动：**

#### 1.Activity super.onCreate执行完毕 ####

Fragment onAttach

Fragment onCreate 

Fragment onCreateView

Fragment onViewCreate

#### 2.Activity super.onStart执行... ####

Fragment onActivityCreated

Fragment onStart

#### 3.Activity super.onResume执行完毕 ####

Fragment onResume


**暂停：**

#### 4.Activity super.onPause执行中... ####

Fragment onPause

Activity super.onPause执行完毕
Activity super.onSaveInstanceState执行...

Fragment onSaveInstanceState

#### 5.Activity super.onStop开始执行...  ####
Fragment onStop

#### 6.Activity super.onDestroy执行中...  ####
Fragment onDestroyView

Fragment onDestroy

Fragment onDetach

**重启：**

#### 7.Activity super.onRestart执行完毕！  ####
Activity super.onStart执行中...

Fragment onStart

#### 8.Activity super.onResume执行完毕！  ####

Fragment onResume

