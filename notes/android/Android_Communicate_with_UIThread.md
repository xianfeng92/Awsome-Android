# Communicate with the UI thread

每个应用程序都有自己的特殊线程来运行UI对象，比如 View objects,这个线程称为UI线程。只有运行在UI线程上的对象才能访问该线程上的其他对象。在线程池中运行的任务(Runnable)是无权访问UI对象。所以我们需要将数据从后台线程移动到UI线程, 此时可以使用 Handler 来完成数据在线程直接的转移.

## Define a handler on the UI thread

Handler 是 Android 系统框架的一部分, 可用于管理线程.一个 Handler 对象接收消息(message)并运行相关代码来处理该消息。我们可以为一个新线程创建一个 Handler,同样也可以创建一个
Handler 关联到已有的线程. Hanlder 中消息的是在其所关联的线程中处理的, 所以当我们创建一个 Handler 关联 UI 线程时, 我们在子线程中往 Handler 中 sendMessage 时, 该 Message的
处理也就会在UI中.

在创建线程池的类的构造过程中实例化 Handler 对象, 并将该对象存储在全局变量中,通过 Handler(Looper) 构造函数将其关联到 UI 线程, 该构造函数使用了一个 looper 对象,这是Android系统线程管理框架的另一部分。当我们基于一个 Looper 来实例化一个 Handler 对象时,该 Handler 就会和 Looper 在相同的线程中.

```
private PhotoManager() {
...
    // Defines a Handler object that's attached to the UI thread
    handler = new Handler(Looper.getMainLooper()) {
    ...
```

在 Handler 内部,我们重写其 handleMessage 方法. 当 Android 系统收到其管理的线程发出的一个新 message 时, 其会根据 message 中的 target 找到目标 Handler,然后回调其 handleMessage 方法. 

__all of the Handler objects for a particular thread receive the same message__. 指定线程的所有Handler对象都会收到这个 message ??? 暂时不理解~~


```
        /*
         * handleMessage() defines the operations to perform when
         * the Handler receives a new Message to process.
         */
        @Override
        public void handleMessage(Message inputMessage) {
            // Gets the image task from the incoming Message object.
            PhotoTask photoTask = (PhotoTask) inputMessage.obj;
            ...
        }
    ...
    }
}
```


## Move data from a task to the UI thread

要将数据从后台线程运行的任务(Runnable)中移动到UI线程中. 首先在任务(Runnable)对象中存储了对数据和UI对象的引用。然后,将任务对象和状态代码传递给 Handler 对象。在此对象中, 向 Handler 发送包含状态和任务对象的消息。因为 Handler 在UI线程上运行，所以它可以更新 UI.

### Store data in the task object

例如,有一个 Runnable 在后台线程上运行, 该 Runnable 主要是 decodes a Bitmap 并将其存储到 PhotoTask , 同时 Runnable 还存储状态代码 decode_state_completed。

```
// A class that decodes photo files into Bitmaps
class PhotoDecodeRunnable implements Runnable {
    ...
    PhotoDecodeRunnable(PhotoTask downloadTask) {
        photoTask = downloadTask;
    }
    ...
    // Gets the downloaded byte array
    byte[] imageBuffer = photoTask.getByteBuffer();
    ...
    // Runs the code for this task
    public void run() {
        ...
        // Tries to decode the image buffer
        returnBitmap = BitmapFactory.decodeByteArray(
                imageBuffer,
                0,
                imageBuffer.length,
                bitmapOptions
        );
        ...
        // Sets the ImageView Bitmap
        photoTask.setImage(returnBitmap);
        // Reports a status of "completed"
        photoTask.handleDecodeState(DECODE_STATE_COMPLETED);
        ...
    }
    ...
}
...
```

PhotoTask 包含了用于显示 Bitmap 的 ImageView 的句柄。即使此时 bitMap 和 ImageView 的引用位于同一对象中, 仍然无用用其更新 ImageView, 因为当前它们并没有在UI线程上运行。


### Send status up the object hierarchy

PhotoTask 是层次结构中的一个更高的对象。它维护对解码数据(bitMao)和将显示数据的视图对象的引用。它从 PhotoDecodeRunnable 接收状态代码,并将其传递给负责维护线程池以及实例化 Hanlder 的一个对象.

```
public class PhotoTask {
    ...
    // Gets a handle to the object that creates the thread pools
    photoManager = PhotoManager.getInstance();
    ...
    public void handleDecodeState(int state) {
        int outState;
        // Converts the decode state to the overall state.
        switch(state) {
            case PhotoDecodeRunnable.DECODE_STATE_COMPLETED:
                outState = PhotoManager.TASK_COMPLETE;
                break;
            ...
        }
        ...
        // Calls the generalized state method
        handleState(outState);
    }
    ...
    // Passes the state to PhotoManager
    void handleState(int state) {
        /*
         * Passes a handle to this task and the
         * current state to the class that created
         * the thread pools
         */
        photoManager.handleState(this, state);
    }
    ...
}
```


### Move data to the UI

photoManager 对象接收状态代码和 PhotoTask 对象的句柄。由于状态为“任务完成”,因此创建包含状态和任务对象的消息并将其发送给 Hander:

```
public class PhotoManager {
    ...
    // Handle status messages from tasks
    public void handleState(PhotoTask photoTask, int state) {
        switch (state) {
            ...
            // The task finished downloading and decoding the image
            case TASK_COMPLETE:
                /*
                 * Creates a message for the Handler
                 * with the state and the task object
                 */
                Message completeMessage =
                        handler.obtainMessage(state, photoTask);
                completeMessage.sendToTarget();
                break;
            ...
        }
        ...
    }
```



最后, Handler.handleMessage() 会检查每个传入消息的状态代码。如果状态代码为“任务完成”,则任务完成. 消息中的 PhotoTask 对象同时包含 bitMap 和 imageView。由于 handleMessage（）是在UI线程上运行, 因此它可以安全地更新UI.

```
    private PhotoManager() {
        ...
            handler = new Handler(Looper.getMainLooper()) {
                @Override
                public void handleMessage(Message inputMessage) {
                    // Gets the task from the incoming Message object.
                    PhotoTask photoTask = (PhotoTask) inputMessage.obj;
                    // Gets the ImageView for this task
                    PhotoView localView = photoTask.getPhotoView();
                    ...
                    switch (inputMessage.what) {
                        ...
                        // The decoding is done
                        case TASK_COMPLETE:
                            /*
                             * Moves the Bitmap from the task
                             * to the View
                             */
                            localView.setImageBitmap(photoTask.getImage());
                            break;
                        ...
                        default:
                            /*
                             * Pass along other messages from the UI
                             */
                            super.handleMessage(inputMessage);
                    }
                    ...
                }
                ...
            }
            ...
    }
...
}
```


[Communicate with the UI thread](https://developer.android.google.cn/training/multiple-threads/communicate-ui.html)
