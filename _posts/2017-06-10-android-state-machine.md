---
layout: post
title: "Android State Machine"
date: 2017-06-10
---

# Android 状态机

官方源码中的demo对Android状态机的工作方式和使用方法有了非常清晰的描述。如果对状态机的工作方式还不清楚的，强烈推荐查看demo，在本文的底部也复制了该demo供查看。

实际上，如果你读懂了demo，Android状态机的原理是显而易见的。本文只是记录了本人阅读源码的一些总结，仅此而已。

## 构造状态机

StateMachine的构造函数都是protected类型，不能实例化，需由其子类进行初始化操作。StateMachine有三个重载的构造函数，可以通过指定消息循环对象或者Handler对象来构造状态机，也可以启动一个消息循环线程来构造状态机。

```java
protected StateMachine(String name) {
  mSmThread = new HandlerThread(name);
  mSmThread.start();
  Looper looper = mSmThread.getLooper();
  initStateMachine(name, looper);
}
protected StateMachine(String name, Looper looper) {
    initStateMachine(name, looper);
}
protected StateMachine(String name, Handler handler) {
  initStateMachine(name, handler.getLooper());
}

//构造一个名为mName的状态机，包含mSmHandler对象，SmHandler继承Handle类，用于派发消息。
private void initStateMachine(String name, Looper looper) {
  mName = name;
  mSmHandler = new SmHandler(looper, this);
}
```

## 建立树形层次结构状态机

```java
protected final void addState(State state, State parent) {
        mSmHandler.addState(state, parent);
}
private final StateInfo addState(State state, State parent) {
  StateInfo parentStateInfo = null;
  if (parent != null) {
    parentStateInfo = mStateInfo.get(parent);
    if (parentStateInfo == null) {
      // Recursively add our parent as it's not been added yet.
      parentStateInfo = addState(parent, null);
    }
  }
  StateInfo stateInfo = mStateInfo.get(state);
  if (stateInfo == null) {
    stateInfo = new StateInfo();
    mStateInfo.put(state, stateInfo);
  }

  // Validate that we aren't adding the same state in two different
  // hierarchies.
  //例如:addstate(a1, p1); addstate(a1,p2)时，a1同时是p1和p2的子状态
  if ((stateInfo.parentStateInfo != null)
      && (stateInfo.parentStateInfo != parentStateInfo)) {
    throw new RuntimeException("state already added");//跑出运行时异常，退出线程
  }
  stateInfo.state = state;
  stateInfo.parentStateInfo = parentStateInfo;
  //初始的状态都是未激活，只有当"enter（）"某个状态后，active才置为true。
  stateInfo.active = false; 
  if (mDbg)
    mSm.log("addStateInternal: X stateInfo: " + stateInfo);
  return stateInfo;
}
```

## 设置状态机初始状态

```java
private final void setInitialState(State initialState) {
mInitialState = initialState;
}
```

## 启动状态机

```java
public void start() {
    // mSmHandler can be null if the state machine has quit.
    SmHandler smh = mSmHandler;
    if (smh == null)
        return;
    smh.completeConstruction();
}
        private final void completeConstruction() {
            //确定状态机层次的深度
            int maxDepth = 0;
            for (StateInfo si : mStateInfo.values()) {
                int depth = 0;
                for (StateInfo i = si; i != null; depth++) {
                    i = i.parentStateInfo;
                }
                if (maxDepth < depth) {
                    maxDepth = depth;
                }
            }
            mStateStack = new StateInfo[maxDepth];
            mTempStateStack = new StateInfo[maxDepth];
            setupInitialStateStack();
            //发送SM_INIT_CMD消息，异步调用enter()
            sendMessageAtFrontOfQueue(obtainMessage(SM_INIT_CMD, mSmHandlerObj));
        }        

        //根据mInitialState值初始化状态栈
        private final void setupInitialStateStack() {
            StateInfo curStateInfo = mStateInfo.get(mInitialState);
            for (mTempStateStackCount = 0; curStateInfo != null; mTempStateStackCount++) {
                mTempStateStack[mTempStateStackCount] = curStateInfo;
                curStateInfo = curStateInfo.parentStateInfo;
            }
            // Empty the StateStack
            mStateStackTopIndex = -1;
            //mTempStateStackCount的顺序是从子状态到父状态排序，反序移动到mStateStack中，因此mStateStack的排序是从父状态到子状态。
            //这个方法在切换状态的时候还会调用，后面会讲到
            moveTempStateStackToStateStack();
        }
```

全局变量mTempStateStackCount为栈的状态个数，mStateStackTopIndex为栈的索引，浏览代码的时候注意区分。

处理SM_INIT_CMD：

```java
                } else if (!mIsConstructionCompleted
                        && (mMsg.what == SM_INIT_CMD)
                        && (mMsg.obj == mSmHandlerObj)) {
                   //只初始化一次状态机
                    mIsConstructionCompleted = true;
                    invokeEnterMethods(0);
                }
```

循环调用从父状态到子状态的enter（）函数，并置为激活状态：

```java
private final void invokeEnterMethods(int stateStackEnteringIndex) {
  for (int i = stateStackEnteringIndex; i <= mStateStackTopIndex; i++) {
    mStateStack[i].state.enter();
    mStateStack[i].active = true;
  }
}
```

## 状态切换

```java
private final void transitionTo(IState destState) {
    mDestState = (State) destState;
}
```

该方法设置了目标状态的值，真正的状态切换时机是在每次处理消息循环时调用performTransitions中执行。此处的含义是如果我们在子类processMessage方法中调用了transitionTo方法，那么在当前消息循环处理完成前，必定会在performTransitions中完成状态切换。

```java
public final void handleMessage(Message msg) {
    if (!mHasQuit) {
        /** Save the current message */
        mMsg = msg;

        /** State that processed the message */
        State msgProcessedState = null;
        if (mIsConstructionCompleted) { //true
            msgProcessedState = processMsg(msg);
        } else if (!mIsConstructionCompleted
                && (mMsg.what == SM_INIT_CMD)
                && (mMsg.obj == mSmHandlerObj)) {
            mIsConstructionCompleted = true;
            invokeEnterMethods(0);
        } else {
            throw new RuntimeException("StateMachine.handleMessage: "
                    + "The start method not called, received msg: "
                    + msg);
        }
        performTransitions(msgProcessedState, msg);

        if (mDbg && mSm != null)
            mSm.log("handleMessage: X");
    }
}
//从子状态到父状态依次处理消息，直到processMessage返回true为止。
private final State processMsg(Message msg) {
  StateInfo curStateInfo = mStateStack[mStateStackTopIndex];
  if (isQuit(msg)) {
    transitionTo(mQuittingState);
  } else {
    while (!curStateInfo.state.processMessage(msg)) {
      curStateInfo = curStateInfo.parentStateInfo;
      if (curStateInfo == null) {
        mSm.unhandledMessage(msg);
        break;
      }
    }
  }
  return (curStateInfo != null) ? curStateInfo.state : null;
}
```

切换状态的规则是：以目标状态为起点，反向查找它的父状态，直到找到已激活状态commonStateInfo停止（不包括），或者父状态为null值。弹出状态栈中从栈顶到commonStateInfo之间的状态，然后将找到的未激活状态以父状态到子状态的顺序填充状态栈，最后激活状态栈中暂未激活的状态。

以Demo中的状态机为例，假设当前状态栈为 {P1, S2},目标状态为S1,那么弹出S2,填充S1,最终状态栈为{P1,S1}。

```java
        private void performTransitions(State msgProcessedState, Message msg) {
            /**
             * 此处省略一些代码，主要目的是记录前后状态信息，在dump信息时用到
             */

            State destState = mDestState;
            if (destState != null) {
                while (true) {
                    //根据destState建立mTempStateStack
                    StateInfo commonStateInfo = setupTempStateStackWithStatesToEnter(destState);
                    //弹出状态栈，并推出状态
                    invokeExitMethods(commonStateInfo);
                    int stateStackEnteringIndex = moveTempStateStackToStateStack();
                    //填充状态栈，并激活状态
                    invokeEnterMethods(stateStackEnteringIndex);
                    //在statemachine的子类中可以将消息加入延迟处理列表，也就是说只有当状态切换完成后，延迟列表中的消息才会被处理。此方法将延迟的消息移动到消息队列的前面，以便快速得到处理。
                    //一般在子类中调用deferMessage方法添加延迟消息到列表中。
                    moveDeferredMessageAtFrontOfQueue();

                    if (destState != mDestState) {
                        // A new mDestState so continue looping
                        destState = mDestState;
                    } else {
                        break;
                    }
                }
                mDestState = null;
            }

            if (destState != null) {
                if (destState == mQuittingState) {
                    mSm.onQuitting();
                    cleanupAfterQuitting();
                } else if (destState == mHaltingState) {
                    mSm.onHalting();
                }
            }
        }
```

相关的方法实现如下：

```java
private final StateInfo setupTempStateStackWithStatesToEnter(
  State destState) {
  mTempStateStackCount = 0;
  StateInfo curStateInfo = mStateInfo.get(destState);
  do {
    mTempStateStack[mTempStateStackCount++] = curStateInfo;
    curStateInfo = curStateInfo.parentStateInfo;
  } while ((curStateInfo != null) && !curStateInfo.active);
  return curStateInfo;
}

private final void invokeExitMethods(StateInfo commonStateInfo) {
  while ((mStateStackTopIndex >= 0)
         && (mStateStack[mStateStackTopIndex] != commonStateInfo)) {
    State curState = mStateStack[mStateStackTopIndex].state;
    curState.exit();
    mStateStack[mStateStackTopIndex].active = false;
    mStateStackTopIndex -= 1;
  }
}

private final int moveTempStateStackToStateStack() {
  int startingIndex = mStateStackTopIndex + 1;
  int i = mTempStateStackCount - 1;
  int j = startingIndex;
  while (i >= 0) {
    mStateStack[j] = mTempStateStack[i];
    j += 1;
    i -= 1;
  }
  mStateStackTopIndex = j - 1;
  return startingIndex;
}

private final void invokeEnterMethods(int stateStackEnteringIndex) {
  for (int i = stateStackEnteringIndex; i <= mStateStackTopIndex; i++) {
    mStateStack[i].state.enter();
    mStateStack[i].active = true;
  }
}

private final void moveDeferredMessageAtFrontOfQueue() {
  for (int i = mDeferredMessages.size() - 1; i >= 0; i--) {
    Message curMsg = mDeferredMessages.get(i);
    sendMessageAtFrontOfQueue(curMsg);
  }
  mDeferredMessages.clear();
}
```



## Demo:

```
/**

 * {@hide}
    *
 * <p>
 * The state machine defined here is a hierarchical state machine which
 * processes messages and can have states arranged hierarchically.
 * </p>
    *
 * <p>
 * A state is a <code>State</code> object and must implement
 * <code>processMessage</code> and optionally <code>enter/exit/getName</code>.
 * The enter/exit methods are equivalent to the construction and destruction in
 * Object Oriented programming and are used to perform initialization and
 * cleanup of the state respectively. The <code>getName</code> method returns
 * the name of the state the default implementation returns the class name it
 * may be desirable to have this return the name of the state instance name
 * instead. In particular if a particular state class has multiple instances.
 * </p>
    *
 * <p>
 * When a state machine is created <code>addState</code> is used to build the
 * hierarchy and <code>setInitialState</code> is used to identify which of these
 * is the initial state. After construction the programmer calls
 * <code>start</code> which initializes and starts the state machine. The first
 * action the StateMachine is to the invoke <code>enter</code> for all of the
 * initial state's hierarchy, starting at its eldest parent. The calls to enter
 * will be done in the context of the StateMachines Handler not in the context
 * of the call to start and they will be invoked before any messages are
 * processed. For example, given the simple state machine below mP1.enter will
 * be invoked and then mS1.enter. Finally, messages sent to the state machine
 * will be processed by the current state, in our simple state machine below
 * that would initially be mS1.processMessage.
 * </p>
 * <code>
        mP1
       /   \
      mS2   mS1 ----> initial state
     </code>
 * <p>
 * After the state machine is created and started, messages are sent to a state
 * machine using <code>sendMessage</code> and the messages are created using
 * <code>obtainMessage</code>. When the state machine receives a message the
 * current state's <code>processMessage</code> is invoked. In the above example
 * mS1.processMessage will be invoked first. The state may use
 * <code>transitionTo</code> to change the current state to a new state
 * </p>
    *
 * <p>
 * Each state in the state machine may have a zero or one parent states and if a
 * child state is unable to handle a message it may have the message processed
 * by its parent by returning false or NOT_HANDLED. If a message is never
 * processed <code>unhandledMessage</code> will be invoked to give one last
 * chance for the state machine to process the message.
 * </p>
    *
 * <p>
 * When all processing is completed a state machine may choose to call
 * <code>transitionToHaltingState</code>. When the current
 * <code>processingMessage</code> returns the state machine will transfer to an
 * internal <code>HaltingState</code> and invoke <code>halting</code>. Any
 * message subsequently received by the state machine will cause
 * <code>haltedProcessMessage</code> to be invoked.
 * </p>
    *
 * <p>
 * If it is desirable to completely stop the state machine call
 * <code>quit</code> or <code>quitNow</code>. These will call <code>exit</code>
 * of the current state and its parents, call <code>onQuiting</code> and then
 * exit Thread/Loopers.
 * </p>
    *
 * <p>
 * In addition to <code>processMessage</code> each <code>State</code> has an
 * <code>enter</code> method and
 * <code>exit</exit> method which may be overridden.
 * </p>
    *
 * <p>
 * Since the states are arranged in a hierarchy transitioning to a new state
 * causes current states to be exited and new states to be entered. To determine
 * the list of states to be entered/exited the common parent closest to the
 * current state is found. We then exit from the current state and its parent's
 * up to but not including the common parent state and then enter all of the new
 * states below the common parent down to the destination state. If there is no
 * common parent all states are exited and then the new states are entered.
 * </p>
    *
 * <p>
 * Two other methods that states can use are <code>deferMessage</code> and
 * <code>sendMessageAtFrontOfQueue</code>. The
 * <code>sendMessageAtFrontOfQueue</code> sends a message but places it on the
 * front of the queue rather than the back. The <code>deferMessage</code> causes
 * the message to be saved on a list until a transition is made to a new state.
 * At which time all of the deferred messages will be put on the front of the
 * state machine queue with the oldest message at the front. These will then be
 * processed by the new current state before any other messages that are on the
 * queue or might be added later. Both of these are protected and may only be
 * invoked from within a state machine.
 * </p>
    *
 * <p>
 * To illustrate some of these properties we'll use state machine with an 8
 * state hierarchy:
 * </p>
 * <code>
         mP0
        /   \
       mP1   mS0
      /   \
      mS2   mS1
     /  \    \
    mS3  mS4  mS5  ---> initial state
</code>
 * <p>
 * After starting mS5 the list of active states is mP0, mP1, mS1 and mS5. So the
 * order of calling processMessage when a message is received is mS5, mS1, mP1,
 * mP0 assuming each processMessage indicates it can't handle this message by
 * returning false or NOT_HANDLED.
 * </p>
    *
 * <p>
 * Now assume mS5.processMessage receives a message it can handle, and during
 * the handling determines the machine should change states. It could call
 * transitionTo(mS4) and return true or HANDLED. Immediately after returning
 * from processMessage the state machine runtime will find the common parent,
 * which is mP1. It will then call mS5.exit, mS1.exit, mS2.enter and then
 * mS4.enter. The new list of active states is mP0, mP1, mS2 and mS4. So when
 * the next message is received mS4.processMessage will be invoked.
 * </p>
    *
 * <p>
 * Now for some concrete examples, here is the canonical HelloWorld as a state
 * machine. It responds with "Hello World" being printed to the log for every
 * message.
 * </p>
 * <code>
  class HelloWorld extends StateMachine {
   HelloWorld(String name) {
       super(name);
       addState(mState1);
       setInitialState(mState1);
   }

   public static HelloWorld makeHelloWorld() {
       HelloWorld hw = new HelloWorld("hw");
       hw.start();
       return hw;
   }

   class State1 extends State {
       &#64;Override public boolean processMessage(Message message) {
           log("Hello World");
           return HANDLED;
       }
   }
   State1 mState1 = new State1();
  }

void testHelloWorld() {
    HelloWorld hw = makeHelloWorld();
    hw.sendMessage(hw.obtainMessage());
}
</code>
 * <p>
 * A more interesting state machine is one with four states with two independent
 * parent states.
 * </p>
 * <code>
        mP1      mP2
       /   \
      mS2   mS1
     </code>
 * <p>
 * Here is a description of this state machine using pseudo code.
 * </p>
 * <code>
  state mP1 {
    enter { log("mP1.enter"); }
    exit { log("mP1.exit");  }
    on msg {
        CMD_2 {
            send(CMD_3);
            defer(msg);
            transitonTo(mS2);
            return HANDLED;
        }
        return NOT_HANDLED;
    }
  }

INITIAL
state mS1 parent mP1 {
     enter { log("mS1.enter"); }
     exit  { log("mS1.exit");  }
     on msg {
         CMD_1 {
             transitionTo(mS1);
             return HANDLED;
         }
         return NOT_HANDLED;
     }
}

state mS2 parent mP1 {
     enter { log("mS2.enter"); }
     exit  { log("mS2.exit");  }
     on msg {
         CMD_2 {
             send(CMD_4);
             return HANDLED;
         }
         CMD_3 {
             defer(msg);
             transitionTo(mP2);
             return HANDLED;
         }
         return NOT_HANDLED;
     }
}

state mP2 {
     enter {
         log("mP2.enter");
         send(CMD_5);
     }
     exit { log("mP2.exit"); }
     on msg {
         CMD_3, CMD_4 { return HANDLED; }
         CMD_5 {
             transitionTo(HaltingState);
             return HANDLED;
         }
         return NOT_HANDLED;
     }
}
</code>
 * <p>
 * The implementation is below and also in StateMachineTest:
 * </p>
 * <code>
  class Hsm1 extends StateMachine {
   public static final int CMD_1 = 1;
   public static final int CMD_2 = 2;
   public static final int CMD_3 = 3;
   public static final int CMD_4 = 4;
   public static final int CMD_5 = 5;

   public static Hsm1 makeHsm1() {
       log("makeHsm1 E");
       Hsm1 sm = new Hsm1("hsm1");
       sm.start();
       log("makeHsm1 X");
       return sm;
   }

   Hsm1(String name) {
       super(name);
       log("ctor E");
      
       // Add states, use indentation to show hierarchy
       addState(mP1);
           addState(mS1, mP1);
           addState(mS2, mP1);
       addState(mP2);
      
       // Set the initial state
       setInitialState(mS1);
       log("ctor X");
   }

   class P1 extends State {
       &#64;Override public void enter() {
           log("mP1.enter");
       }
       &#64;Override public boolean processMessage(Message message) {
           boolean retVal;
           log("mP1.processMessage what=" + message.what);
           switch(message.what) {
           case CMD_2:
               // CMD_2 will arrive in mS2 before CMD_3
               sendMessage(obtainMessage(CMD_3));
               deferMessage(message);
               transitionTo(mS2);
               retVal = HANDLED;
               break;
           default:
               // Any message we don't understand in this state invokes unhandledMessage
               retVal = NOT_HANDLED;
               break;
           }
           return retVal;
       }
       &#64;Override public void exit() {
           log("mP1.exit");
       }
   }

   class S1 extends State {
       &#64;Override public void enter() {
           log("mS1.enter");
       }
       &#64;Override public boolean processMessage(Message message) {
           log("S1.processMessage what=" + message.what);
           if (message.what == CMD_1) {
               // Transition to ourself to show that enter/exit is called
               transitionTo(mS1);
               return HANDLED;
           } else {
               // Let parent process all other messages
               return NOT_HANDLED;
           }
       }
       &#64;Override public void exit() {
           log("mS1.exit");
       }
   }

   class S2 extends State {
       &#64;Override public void enter() {
           log("mS2.enter");
       }
       &#64;Override public boolean processMessage(Message message) {
           boolean retVal;
           log("mS2.processMessage what=" + message.what);
           switch(message.what) {
           case(CMD_2):
               sendMessage(obtainMessage(CMD_4));
               retVal = HANDLED;
               break;
           case(CMD_3):
               deferMessage(message);
               transitionTo(mP2);
               retVal = HANDLED;
               break;
           default:
               retVal = NOT_HANDLED;
               break;
           }
           return retVal;
       }
       &#64;Override public void exit() {
           log("mS2.exit");
       }
   }

   class P2 extends State {
       &#64;Override public void enter() {
           log("mP2.enter");
           sendMessage(obtainMessage(CMD_5));
       }
       &#64;Override public boolean processMessage(Message message) {
           log("P2.processMessage what=" + message.what);
           switch(message.what) {
           case(CMD_3):
               break;
           case(CMD_4):
               break;
           case(CMD_5):
               transitionToHaltingState();
               break;
           }
           return HANDLED;
       }
       &#64;Override public void exit() {
           log("mP2.exit");
       }
   }

   &#64;Override
   void onHalting() {
       log("halting");
       synchronized (this) {
           this.notifyAll();
       }
   }

   P1 mP1 = new P1();
   S1 mS1 = new S1();
   S2 mS2 = new S2();
   P2 mP2 = new P2();
  }
  </code>
 * <p>
 * If this is executed by sending two messages CMD_1 and CMD_2 (Note the
 * synchronize is only needed because we use hsm.wait())
 * </p>
 * <code>
  Hsm1 hsm = makeHsm1();
  synchronize(hsm) {
    hsm.sendMessage(obtainMessage(hsm.CMD_1));
    hsm.sendMessage(obtainMessage(hsm.CMD_2));
    try {
         // wait for the messages to be handled
         hsm.wait();
    } catch (InterruptedException e) {
         loge("exception while waiting " + e.getMessage());
    }
  }
  </code>
 * <p>
 * The output is:
 * </p>
 * <code>
  D/hsm1    ( 1999): makeHsm1 E
  D/hsm1    ( 1999): ctor E
  D/hsm1    ( 1999): ctor X
  D/hsm1    ( 1999): mP1.enter
  D/hsm1    ( 1999): mS1.enter
  D/hsm1    ( 1999): makeHsm1 X
  D/hsm1    ( 1999): mS1.processMessage what=1
  D/hsm1    ( 1999): mS1.exit
  D/hsm1    ( 1999): mS1.enter
  D/hsm1    ( 1999): mS1.processMessage what=2
  D/hsm1    ( 1999): mP1.processMessage what=2
  D/hsm1    ( 1999): mS1.exit
  D/hsm1    ( 1999): mS2.enter
  D/hsm1    ( 1999): mS2.processMessage what=2
  D/hsm1    ( 1999): mS2.processMessage what=3
  D/hsm1    ( 1999): mS2.exit
  D/hsm1    ( 1999): mP1.exit
  D/hsm1    ( 1999): mP2.enter
  D/hsm1    ( 1999): mP2.processMessage what=3
  D/hsm1    ( 1999): mP2.processMessage what=4
  D/hsm1    ( 1999): mP2.processMessage what=5
  D/hsm1    ( 1999): mP2.exit
  D/hsm1    ( 1999): halting
  </code>
   */
```
