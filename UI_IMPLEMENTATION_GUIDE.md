# Android UI Implementation Guide

## Table of Contents
1. [Overview](#overview)
2. [Common UI Patterns](#common-ui-patterns)
3. [ChatApp - Conversational Interface](#chatapp---conversational-interface)
4. [ImageClassification - Static Image Analysis](#imageclassification---static-image-analysis)
5. [ObjectDetection - Real-time Camera with Overlays](#objectdetection---real-time-camera-with-overlays)
6. [SemanticSegmentation - Real-time Scene Understanding](#semanticsegmentation---real-time-scene-understanding)
7. [SuperResolution - Image Enhancement](#superresolution---image-enhancement)
8. [Custom Views and Rendering](#custom-views-and-rendering)
9. [Threading Architecture](#threading-architecture)
10. [Themes and Styling](#themes-and-styling)
11. [Best Practices](#best-practices)

---

## Overview

This guide documents the UI implementation patterns across all Android sample apps in the Qualcomm AI Hub project. Each app demonstrates different UI paradigms optimized for specific AI use cases:

| App | UI Pattern | Key Components |
|-----|------------|----------------|
| **ChatApp** | Conversational Interface | RecyclerView, Chat Bubbles, Streaming Text |
| **ImageClassification** | Static Image Analysis | Spinner, CardView, RadioGroup |
| **ObjectDetection** | Real-time Camera | Camera2 API, Custom Canvas Drawing, Bounding Boxes |
| **SemanticSegmentation** | Real-time Camera | Camera2 API, Segmentation Overlay |
| **SuperResolution** | Image Enhancement | Image Before/After Display |

### Common Material Design Components
- `ConstraintLayout` - Responsive layouts
- `CardView` - Elevated containers for visual hierarchy
- `RecyclerView` - Efficient list rendering
- `Spinner` - Dropdown selections
- `RadioButton/RadioGroup` - Delegate selection (NPU/GPU/CPU)
- `Button` - Action triggers with Material ripple effects
- `TextureView` - Camera preview rendering

---

## Common UI Patterns

### 1. Activity Structure

All apps follow a consistent activity structure:

```java
public class MainActivity extends AppCompatActivity {
    // UI Elements
    private Button actionButton;
    private TextView resultText;

    // Threading
    private ExecutorService backgroundExecutor;
    private Handler mainLooperHandler;

    // Model wrapper
    private ModelWrapper model;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main_activity);

        // Initialize threading
        backgroundExecutor = Executors.newSingleThreadExecutor();
        mainLooperHandler = new Handler(Looper.getMainLooper());

        // Initialize UI
        initializeUI();

        // Load model asynchronously
        createModelAsync();
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (model != null) {
            model.close();
        }
        backgroundExecutor.shutdown();
    }
}
```

### 2. Delegate Selection Pattern

All apps provide hardware acceleration options:

**XML Layout**:
```xml
<RadioGroup
    android:id="@+id/delegateSelectionGroup"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="horizontal">

    <RadioButton
        android:id="@+id/cpuOnlyRadio"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="CPU Only"
        android:buttonTint="@color/purple_qcom" />

    <RadioButton
        android:id="@+id/defaultDelegateRadio"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="All Hardware"
        android:checked="true"
        android:buttonTint="@color/purple_qcom" />
</RadioGroup>
```

**Java Implementation**:
```java
RadioGroup delegateGroup = findViewById(R.id.delegateSelectionGroup);
delegateGroup.setOnCheckedChangeListener((group, checkedId) -> {
    if (checkedId == R.id.cpuOnlyRadio) {
        useCPUOnly = true;
    } else {
        useCPUOnly = false;
    }
    clearResults();
});
```

### 3. Performance Metrics Display

All apps show timing information:

```xml
<TextView
    android:id="@+id/inferenceTimeText"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="Inference Time"
    android:textSize="18sp" />

<TextView
    android:id="@+id/inferenceTimeResultText"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="-- ms"
    android:textColor="@color/purple_qcom"
    android:textSize="18sp" />
```

**Update Pattern**:
```java
void updatePerformanceMetrics(long preprocessTime, long inferenceTime, long postprocessTime) {
    long totalTime = preprocessTime + inferenceTime + postprocessTime;

    mainLooperHandler.post(() -> {
        inferenceTimeView.setText(inferenceTime + " ms");
        predictionTimeView.setText(totalTime + " ms");
    });
}
```

### 4. UI State Management

Consistent pattern for enabling/disabling UI during inference:

```java
void setInferenceUIEnabled(boolean enabled) {
    actionButton.setEnabled(enabled);
    actionButton.setAlpha(enabled ? 1.0f : 0.5f);
    imageSelector.setEnabled(enabled);
    delegateGroup.setEnabled(enabled);

    if (!enabled) {
        resultText.setText("Processing...");
    }
}
```

---

## ChatApp - Conversational Interface

### Layout Architecture

**Main Chat Layout** (`chat.xml`):

```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <!-- Message List -->
    <RelativeLayout
        android:id="@+id/toolbar"
        android:layout_above="@id/bottom_layout"
        android:padding="10dp">

        <androidx.recyclerview.widget.RecyclerView
            android:id="@+id/chat_recycler_view"
            android:stackFromBottom="true"
            android:layout_width="410dp"
            android:layout_height="wrap_content" />
    </RelativeLayout>

    <!-- Input Area -->
    <RelativeLayout
        android:id="@+id/bottom_layout"
        android:layout_alignParentBottom="true"
        android:padding="8dp">

        <EditText
            android:id="@+id/user_input"
            android:layout_toStartOf="@id/send_button"
            android:background="@drawable/text_rounded_corner"
            android:hint="What's on your mind?"
            android:padding="10dp" />

        <ImageButton
            android:id="@+id/send_button"
            android:layout_alignParentEnd="true"
            android:src="@android:drawable/ic_menu_send" />
    </RelativeLayout>
</RelativeLayout>
```

**Key Features**:
- `stackFromBottom="true"` - Ensures chat scrolls from bottom like messaging apps
- `layout_above` and `layout_alignParentBottom` - Keeps input fixed at bottom
- Rounded corner EditText for modern appearance

### Chat Message Layout

**Message Row** (`chat_row.xml`):

```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="8dp">

    <!-- Bot Message (Left) -->
    <LinearLayout
        android:id="@+id/left_chat_layout"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:background="@drawable/bot_response"
        android:padding="8dp">

        <TextView
            android:id="@+id/bot_message"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textSize="20sp" />
    </LinearLayout>

    <!-- User Message (Right) -->
    <LinearLayout
        android:id="@+id/right_chat_layout"
        android:layout_alignParentEnd="true"
        android:background="@drawable/user_input"
        android:padding="8dp">

        <TextView
            android:id="@+id/user_message"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:background="@color/user_color"
            android:textSize="20sp" />
    </LinearLayout>
</RelativeLayout>
```

### Conversation Activity Implementation

**File**: `apps/android/ChatApp/src/main/java/com/quicinc/chatapp/Conversation.java`

```java
public class Conversation extends AppCompatActivity {
    ArrayList<ChatMessage> messages = new ArrayList<>(1000);
    private static final String WELCOME_MESSAGE = "Hi! How can I help you?";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.chat);

        // Setup RecyclerView
        RecyclerView recyclerView = findViewById(R.id.chat_recycler_view);
        Message_RecyclerViewAdapter adapter = new Message_RecyclerViewAdapter(this, messages);
        recyclerView.setAdapter(adapter);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));

        // UI Elements
        ImageButton sendButton = findViewById(R.id.send_button);
        TextView userInput = findViewById(R.id.user_input);

        // Load Genie model
        String modelDir = getModelDirectory();
        String htpConfig = getHtpConfigPath();
        GenieWrapper genieWrapper = new GenieWrapper(modelDir, htpConfig);

        // Add welcome message
        messages.add(new ChatMessage(WELCOME_MESSAGE, MessageSender.BOT));

        // Send button handler
        sendButton.setOnClickListener(view -> {
            String userMessage = userInput.getText().toString();
            userInput.setText("");

            // Add user message to UI
            adapter.addMessage(new ChatMessage(userMessage, MessageSender.USER));
            adapter.notifyItemInserted(adapter.getItemCount() - 1);

            int botResponseIndex = adapter.getItemCount();
            recyclerView.smoothScrollToPosition(botResponseIndex);

            // Run inference in background
            ExecutorService service = Executors.newSingleThreadExecutor();
            service.execute(() -> {
                genieWrapper.getResponseForPrompt(userMessage, new StringCallback() {
                    @Override
                    public void onNewString(String token) {
                        runOnUiThread(() -> {
                            // Update bot message with streaming tokens
                            adapter.updateBotMessage(token);
                            adapter.notifyItemChanged(botResponseIndex);
                        });
                    }
                });
            });

            recyclerView.scrollToPosition(adapter.getItemCount() - 1);
        });
    }
}
```

### Custom RecyclerView Adapter

**File**: `apps/android/ChatApp/src/main/java/com/quicinc/chatapp/Message_RecyclerViewAdapter.java`

```java
public class Message_RecyclerViewAdapter
    extends RecyclerView.Adapter<Message_RecyclerViewAdapter.MyViewHolder> {

    private ArrayList<ChatMessage> messages;
    private Context context;

    public static class MyViewHolder extends RecyclerView.ViewHolder {
        TextView userMessage;
        TextView botMessage;
        LinearLayout leftChatLayout;
        LinearLayout rightChatLayout;

        public MyViewHolder(@NonNull View itemView) {
            super(itemView);
            userMessage = itemView.findViewById(R.id.user_message);
            botMessage = itemView.findViewById(R.id.bot_message);
            leftChatLayout = itemView.findViewById(R.id.left_chat_layout);
            rightChatLayout = itemView.findViewById(R.id.right_chat_layout);
        }
    }

    @Override
    public void onBindViewHolder(@NonNull MyViewHolder holder, int position) {
        ChatMessage msg = messages.get(position);

        if (msg.isMessageFromUser()) {
            // Show user message on right
            holder.userMessage.setText(msg.getMessage());
            holder.leftChatLayout.setVisibility(View.GONE);
            holder.rightChatLayout.setVisibility(View.VISIBLE);
        } else {
            // Show bot message on left
            holder.botMessage.setText(msg.getMessage());
            holder.leftChatLayout.setVisibility(View.VISIBLE);
            holder.rightChatLayout.setVisibility(View.GONE);
        }
    }

    // Support for streaming responses
    public void updateBotMessage(String newToken) {
        if (messages.isEmpty()) {
            messages.add(new ChatMessage(newToken, MessageSender.BOT));
        } else {
            ChatMessage lastMessage = messages.get(messages.size() - 1);
            if (!lastMessage.isMessageFromUser()) {
                // Append to existing bot message
                lastMessage.appendMessage(newToken);
            } else {
                // Create new bot message
                messages.add(new ChatMessage(newToken, MessageSender.BOT));
            }
        }
    }

    public void addMessage(ChatMessage message) {
        messages.add(message);
    }

    @Override
    public int getItemCount() {
        return messages.size();
    }
}
```

### Chat Message Data Model

```java
public class ChatMessage {
    String message;
    MessageSender sender;

    public ChatMessage(String message, MessageSender sender) {
        this.message = message;
        this.sender = sender;
    }

    public boolean isMessageFromUser() {
        return sender == MessageSender.USER;
    }

    public void appendMessage(String token) {
        this.message += token;
    }

    public String getMessage() {
        return message;
    }
}

enum MessageSender {
    USER,
    BOT
}
```

### Drawable Resources

**Bot Response Bubble** (`bot_response.xml`):
```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android">
    <solid android:color="#9A9C27B0" />
    <corners android:radius="15dp" />
</shape>
```

**User Input Bubble** (`user_input.xml`):
```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android">
    <solid android:color="#2196F3" />
    <corners android:radius="15dp" />
</shape>
```

**Text Input Background** (`text_rounded_corner.xml`):
```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android">
    <solid android:color="#FFFFFF" />
    <corners android:radius="10dp" />
    <stroke android:width="1dp" android:color="#CCCCCC" />
</shape>
```

---

## ImageClassification - Static Image Analysis

### Layout Structure

**File**: `apps/android/ImageClassification/src/main/res/layout/main_activity.xml`

```xml
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <!-- Title Header -->
    <androidx.cardview.widget.CardView
        android:id="@+id/appTitleCard"
        android:layout_width="409sp"
        android:layout_height="50sp"
        android:backgroundTint="@color/purple_qcom"
        app:layout_constraintTop_toTopOf="parent">

        <TextView
            android:id="@+id/appTitle"
            android:layout_gravity="center"
            android:text="Image Classification"
            android:textColor="@color/white"
            android:textSize="20sp"
            android:textStyle="bold" />
    </androidx.cardview.widget.CardView>

    <!-- Image Display -->
    <androidx.cardview.widget.CardView
        android:id="@+id/selectedImageCard"
        android:layout_width="360sp"
        android:layout_height="360sp"
        android:layout_marginTop="10sp"
        app:layout_constraintTop_toBottomOf="@+id/appTitleCard">

        <ImageView
            android:id="@+id/selectedImageView"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
    </androidx.cardview.widget.CardView>

    <!-- Prediction Result -->
    <TextView
        android:id="@+id/predictionResultText"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textColor="@color/purple_qcom"
        android:textSize="18sp"
        android:textStyle="bold"
        app:layout_constraintTop_toBottomOf="@+id/selectedImageCard" />

    <!-- Timing Metrics -->
    <TextView
        android:id="@+id/inferenceTimeText"
        android:text="Inference Time"
        android:textSize="18sp"
        app:layout_constraintTop_toBottomOf="@+id/predictionResultText" />

    <TextView
        android:id="@+id/inferenceTimeResultText"
        android:text="-- ms"
        android:textColor="@color/purple_qcom"
        android:textSize="18sp" />

    <TextView
        android:id="@+id/predictionTimeText"
        android:text="End-to-End Prediction Time"
        android:textSize="18sp" />

    <TextView
        android:id="@+id/predictionTimeResultText"
        android:text="-- ms"
        android:textColor="@color/purple_qcom"
        android:textSize="18sp" />

    <!-- Run Model Button -->
    <Button
        android:id="@+id/runModelButton"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:backgroundTint="@color/purple_qcom"
        android:text="Run Model"
        android:textColor="@color/white"
        app:layout_constraintBottom_toTopOf="@+id/delegateSelectionGroup" />

    <!-- Delegate Selection -->
    <RadioGroup
        android:id="@+id/delegateSelectionGroup"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        app:layout_constraintBottom_toTopOf="@+id/imageSelectorCard">

        <RadioButton
            android:id="@+id/cpuOnlyRadio"
            android:text="CPU Only"
            android:buttonTint="@color/purple_qcom" />

        <RadioButton
            android:id="@+id/defaultDelegateRadio"
            android:text="All Hardware"
            android:checked="true"
            android:buttonTint="@color/purple_qcom" />
    </RadioGroup>

    <!-- Image Selector -->
    <androidx.cardview.widget.CardView
        android:id="@+id/imageSelectorCard"
        android:layout_width="409sp"
        android:layout_height="50sp"
        android:backgroundTint="@color/purple_qcom"
        app:layout_constraintBottom_toBottomOf="parent">

        <TextView
            android:text="Image"
            android:textColor="@color/white"
            android:textSize="17sp" />

        <Spinner
            android:id="@+id/imageSelector"
            android:layout_gravity="end"
            android:theme="@style/spinnerTheme" />
    </androidx.cardview.widget.CardView>
</androidx.constraintlayout.widget.ConstraintLayout>
```

### MainActivity Implementation

**File**: `apps/android/ImageClassification/src/main/java/com/quicinc/imageclassification/MainActivity.java`

```java
public class MainActivity extends AppCompatActivity {
    // UI Elements
    private ImageView selectedImageView;
    private TextView predictionResultView;
    private TextView inferenceTimeView;
    private TextView predictionTimeView;
    private Spinner imageSelector;
    private Button runModelButton;
    private RadioGroup delegateGroup;

    // Threading
    private ExecutorService backgroundExecutor;
    private Handler mainLooperHandler;

    // Model instances
    private ImageClassification defaultDelegateClassifier;
    private ImageClassification cpuOnlyClassifier;
    private boolean useCPUOnly = false;

    // Current image
    private Bitmap selectedImage;

    // Image selection options
    private final String[] imageSelectorOptions = {
        "Not Selected",
        "Sample1.png",
        "Sample2.png",
        "Sample3.png",
        "From Gallery"
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main_activity);

        // Initialize threading
        backgroundExecutor = Executors.newSingleThreadExecutor();
        mainLooperHandler = new Handler(Looper.getMainLooper());

        // Get UI elements
        selectedImageView = findViewById(R.id.selectedImageView);
        predictionResultView = findViewById(R.id.predictionResultText);
        inferenceTimeView = findViewById(R.id.inferenceTimeResultText);
        predictionTimeView = findViewById(R.id.predictionTimeResultText);
        imageSelector = findViewById(R.id.imageSelector);
        runModelButton = findViewById(R.id.runModelButton);
        delegateGroup = findViewById(R.id.delegateSelectionGroup);

        // Setup image selector
        ArrayAdapter<String> adapter = new ArrayAdapter<>(
            this,
            android.R.layout.simple_spinner_item,
            imageSelectorOptions
        );
        adapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
        imageSelector.setAdapter(adapter);

        // Image selection listener
        imageSelector.setOnItemSelectedListener(new AdapterView.OnItemSelectedListener() {
            @Override
            public void onItemSelected(AdapterView<?> parent, View view,
                                      int position, long id) {
                String selected = (String) parent.getItemAtPosition(position);

                if (selected.equals("From Gallery")) {
                    // Launch gallery picker
                    Intent intent = new Intent();
                    intent.setType("image/*");
                    intent.setAction(Intent.ACTION_GET_CONTENT);
                    galleryLauncher.launch(intent);
                } else if (!selected.equals("Not Selected")) {
                    // Load sample image
                    loadImageFromAssets(selected);
                }
            }

            @Override
            public void onNothingSelected(AdapterView<?> parent) {}
        });

        // Delegate selection listener
        delegateGroup.setOnCheckedChangeListener((group, checkedId) -> {
            useCPUOnly = (checkedId == R.id.cpuOnlyRadio);
            clearPredictionResults();
        });

        // Run model button
        runModelButton.setOnClickListener(view -> {
            if (selectedImage != null) {
                runInferenceAsync();
            }
        });

        // Load models asynchronously
        createModelsAsync();
    }

    // Gallery picker result handler
    private final ActivityResultLauncher<Intent> galleryLauncher =
        registerForActivityResult(
            new ActivityResultContracts.StartActivityForResult(),
            result -> {
                if (result.getResultCode() == RESULT_OK && result.getData() != null) {
                    Uri imageUri = result.getData().getData();
                    loadImageFromUri(imageUri);
                }
            }
        );

    private void createModelsAsync() {
        setInferenceUIEnabled(false);

        backgroundExecutor.execute(() -> {
            try {
                // Load model with all hardware acceleration
                defaultDelegateClassifier = new ImageClassification(
                    this,
                    getString(R.string.tfLiteModelAsset),
                    getString(R.string.tfLiteLabelsAsset),
                    AIHubDefaults.delegatePriorityOrder
                );

                // Load CPU-only model
                cpuOnlyClassifier = new ImageClassification(
                    this,
                    getString(R.string.tfLiteModelAsset),
                    getString(R.string.tfLiteLabelsAsset),
                    AIHubDefaults.delegatePriorityOrderForDelegates(new HashSet<>())
                );

                mainLooperHandler.post(() -> {
                    setInferenceUIEnabled(true);
                    Toast.makeText(this, "Models loaded", Toast.LENGTH_SHORT).show();
                });
            } catch (Exception e) {
                mainLooperHandler.post(() -> {
                    Toast.makeText(this, "Error loading models: " + e.getMessage(),
                                 Toast.LENGTH_LONG).show();
                });
            }
        });
    }

    private void loadImageFromAssets(String imageName) {
        backgroundExecutor.execute(() -> {
            try {
                InputStream stream = getAssets().open("images/" + imageName);
                Bitmap bitmap = BitmapFactory.decodeStream(stream);

                mainLooperHandler.post(() -> {
                    selectedImage = bitmap;
                    selectedImageView.setImageBitmap(bitmap);
                    setInferenceUIEnabled(true);
                });
            } catch (IOException e) {
                mainLooperHandler.post(() -> {
                    Toast.makeText(this, "Error loading image", Toast.LENGTH_SHORT).show();
                });
            }
        });
    }

    private void runInferenceAsync() {
        setInferenceUIEnabled(false);
        predictionResultView.setText("Inferencing...");

        ImageClassification classifier = useCPUOnly ?
            cpuOnlyClassifier : defaultDelegateClassifier;

        backgroundExecutor.execute(() -> {
            // Run prediction
            ArrayList<String> results = classifier.predictClassesFromImage(selectedImage);
            String resultText = String.join(", ", results);

            // Get timing info
            long inferenceTime = classifier.getLastInferenceTime();
            long preprocessTime = classifier.getLastPreprocessingTime();
            long postprocessTime = classifier.getLastPostprocessingTime();
            long totalTime = preprocessTime + inferenceTime + postprocessTime;

            // Update UI
            mainLooperHandler.post(() -> {
                predictionResultView.setText(resultText);
                inferenceTimeView.setText(inferenceTime + " ms");
                predictionTimeView.setText(totalTime + " ms");
                setInferenceUIEnabled(true);
            });
        });
    }

    private void setInferenceUIEnabled(boolean enabled) {
        runModelButton.setEnabled(enabled);
        runModelButton.setAlpha(enabled ? 1.0f : 0.5f);
        imageSelector.setEnabled(enabled);
        delegateGroup.setEnabled(enabled);

        if (!enabled) {
            predictionResultView.setText("");
            inferenceTimeView.setText("-- ms");
            predictionTimeView.setText("-- ms");
        }
    }

    private void clearPredictionResults() {
        predictionResultView.setText("");
        inferenceTimeView.setText("-- ms");
        predictionTimeView.setText("-- ms");
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (defaultDelegateClassifier != null) {
            defaultDelegateClassifier.close();
        }
        if (cpuOnlyClassifier != null) {
            cpuOnlyClassifier.close();
        }
        backgroundExecutor.shutdown();
    }
}
```

---

## ObjectDetection - Real-time Camera with Overlays

### Layout Files

**Main Activity Layout** (`main_activity.xml`):
```xml
<androidx.constraintlayout.widget.ConstraintLayout>
    <FrameLayout
        android:id="@+id/main_content"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <ProgressBar
        android:id="@+id/progressBar"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

**Camera Fragment Layout** (`fragment_camera.xml`):
```xml
<RelativeLayout>
    <!-- Camera Preview (Hidden) -->
    <TextureView
        android:id="@+id/surface"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:alpha="0" />

    <!-- Custom Rendering Overlay -->
    <com.quicinc.objectdetection.FragmentRender
        android:id="@+id/fragment_render"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <ProgressBar
        android:id="@+id/progressBar"
        android:layout_centerInParent="true" />
</RelativeLayout>
```

### MainActivity - Camera Setup

**File**: `apps/android/ObjectDetection/src/main/java/com/quicinc/objectdetection/MainActivity.java`

```java
public class MainActivity extends AppCompatActivity {
    private ObjectDetection detector;
    private ProgressBar progressBar;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main_activity);

        progressBar = findViewById(R.id.progressBar);

        // Check camera permission
        if (checkSelfPermission(Manifest.permission.CAMERA) !=
            PackageManager.PERMISSION_GRANTED) {
            requestPermissions(new String[]{Manifest.permission.CAMERA},
                             CAMERA_PERMISSION_CODE);
        } else {
            loadModelAndStartCamera();
        }
    }

    private void loadModelAndStartCamera() {
        progressBar.setVisibility(View.VISIBLE);

        ExecutorService executor = Executors.newSingleThreadExecutor();
        executor.execute(() -> {
            try {
                // Load object detection model
                detector = new ObjectDetection(
                    this,
                    getString(R.string.tfLiteModelAsset),
                    getString(R.string.tfLiteLabelsAsset),
                    AIHubDefaults.delegatePriorityOrder
                );

                runOnUiThread(() -> {
                    progressBar.setVisibility(View.GONE);
                    launchCameraFragment();
                });
            } catch (Exception e) {
                runOnUiThread(() -> {
                    Toast.makeText(this, "Error: " + e.getMessage(),
                                 Toast.LENGTH_LONG).show();
                    finish();
                });
            }
        });
    }

    private void launchCameraFragment() {
        FragmentTransaction transaction = getSupportFragmentManager().beginTransaction();
        transaction.add(R.id.main_content, CameraFragment.create(detector));
        transaction.commitAllowingStateLoss();
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, String[] permissions,
                                          int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == CAMERA_PERMISSION_CODE) {
            if (grantResults.length > 0 &&
                grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                loadModelAndStartCamera();
            } else {
                Toast.makeText(this, "Camera permission required",
                             Toast.LENGTH_LONG).show();
                finish();
            }
        }
    }
}
```

### CameraFragment - Real-time Inference

**File**: `apps/android/ObjectDetection/src/main/java/com/quicinc/objectdetection/CameraFragment.java`

```java
public class CameraFragment extends Fragment {
    private ObjectDetection detector;
    private TextureView textureView;
    private FragmentRender fragmentRender;
    private CameraDevice cameraDevice;
    private Size previewSize;
    private float fps = 0;
    private long lastFrameTime = 0;

    public static CameraFragment create(ObjectDetection detector) {
        CameraFragment fragment = new CameraFragment();
        fragment.detector = detector;
        return fragment;
    }

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                            Bundle savedInstanceState) {
        return inflater.inflate(R.layout.fragment_camera, container, false);
    }

    @Override
    public void onViewCreated(View view, Bundle savedInstanceState) {
        textureView = view.findViewById(R.id.surface);
        fragmentRender = view.findViewById(R.id.fragment_render);

        textureView.setSurfaceTextureListener(new TextureView.SurfaceTextureListener() {
            @Override
            public void onSurfaceTextureAvailable(SurfaceTexture surface,
                                                 int width, int height) {
                openCamera();
            }

            @Override
            public void onSurfaceTextureSizeChanged(SurfaceTexture surface,
                                                   int width, int height) {}

            @Override
            public boolean onSurfaceTextureDestroyed(SurfaceTexture surface) {
                return true;
            }

            @Override
            public void onSurfaceTextureUpdated(SurfaceTexture surface) {}
        });
    }

    private void openCamera() {
        CameraManager manager = (CameraManager)
            getActivity().getSystemService(Context.CAMERA_SERVICE);

        try {
            String cameraId = manager.getCameraIdList()[0];
            CameraCharacteristics characteristics =
                manager.getCameraCharacteristics(cameraId);

            StreamConfigurationMap map = characteristics.get(
                CameraCharacteristics.SCALER_STREAM_CONFIGURATION_MAP);
            previewSize = map.getOutputSizes(SurfaceTexture.class)[0];

            manager.openCamera(cameraId, new CameraDevice.StateCallback() {
                @Override
                public void onOpened(CameraDevice camera) {
                    cameraDevice = camera;
                    startPreview();
                }

                @Override
                public void onDisconnected(CameraDevice camera) {
                    camera.close();
                }

                @Override
                public void onError(CameraDevice camera, int error) {
                    camera.close();
                }
            }, null);
        } catch (Exception e) {
            Log.e("CameraFragment", "Error opening camera", e);
        }
    }

    private void startPreview() {
        try {
            SurfaceTexture texture = textureView.getSurfaceTexture();
            texture.setDefaultBufferSize(previewSize.getWidth(),
                                        previewSize.getHeight());
            Surface surface = new Surface(texture);

            CaptureRequest.Builder builder = cameraDevice.createCaptureRequest(
                CameraDevice.TEMPLATE_PREVIEW);
            builder.addTarget(surface);

            cameraDevice.createCaptureSession(
                Collections.singletonList(surface),
                new CameraCaptureSession.StateCallback() {
                    @Override
                    public void onConfigured(CameraCaptureSession session) {
                        try {
                            session.setRepeatingRequest(
                                builder.build(),
                                new CaptureCallback(),
                                null
                            );
                        } catch (Exception e) {
                            Log.e("CameraFragment", "Error starting preview", e);
                        }
                    }

                    @Override
                    public void onConfigureFailed(CameraCaptureSession session) {}
                },
                null
            );
        } catch (Exception e) {
            Log.e("CameraFragment", "Error creating preview", e);
        }
    }

    private class CaptureCallback extends CameraCaptureSession.CaptureCallback {
        @Override
        public void onCaptureCompleted(CameraCaptureSession session,
                                      CaptureRequest request,
                                      TotalCaptureResult result) {
            // Calculate FPS
            long currentTime = System.currentTimeMillis();
            if (lastFrameTime != 0) {
                fps = 1000.0f / (currentTime - lastFrameTime);
            }
            lastFrameTime = currentTime;

            // Get frame from TextureView
            Bitmap frame = textureView.getBitmap();

            // Run object detection
            ArrayList<RectangleBox> detections = new ArrayList<>();
            int orientation = getDisplayRotation();
            detector.predict(frame, orientation, detections);

            // Update custom view with results
            fragmentRender.setCoordsList(detections);
            fragmentRender.render(
                frame,
                previewSize,
                fps,
                detector.getLastInferenceTime(),
                detector.getLastPreprocessingTime(),
                detector.getLastPostprocessingTime(),
                orientation
            );
        }
    }

    private int getDisplayRotation() {
        WindowManager windowManager = (WindowManager)
            getActivity().getSystemService(Context.WINDOW_SERVICE);
        int rotation = windowManager.getDefaultDisplay().getRotation();

        switch (rotation) {
            case Surface.ROTATION_0: return 0;
            case Surface.ROTATION_90: return 1;
            case Surface.ROTATION_180: return 2;
            case Surface.ROTATION_270: return 3;
            default: return 0;
        }
    }

    @Override
    public void onPause() {
        super.onPause();
        if (cameraDevice != null) {
            cameraDevice.close();
        }
    }
}
```

---

## Custom Views and Rendering

### FragmentRender - Bounding Box Overlay

**File**: `apps/android/ObjectDetection/src/main/java/com/quicinc/objectdetection/FragmentRender.java`

```java
public class FragmentRender extends View {
    private final ReentrantLock lock = new ReentrantLock();
    private Bitmap bitmap = null;
    private Size cameraSize = null;
    private ArrayList<RectangleBox> boxList = new ArrayList<>();
    private int displayRotation = 0;
    private final Rect targetRect = new Rect();
    private float fps;
    private long inferTime = 0;
    private long preprocessTime = 0;
    private long postprocessTime = 0;
    private Matrix transform = new Matrix();

    // Paint objects
    private final Paint textPaint = new Paint();
    private final Paint framePaint = new Paint();
    private final Paint labelFramePaint = new Paint();

    // 19 distinct colors for different object classes
    public static @ColorInt int labelColor(int label, int alpha) {
        final int[] baseColors = new int[]{
            0xFFF44336, // Red
            0xFFE91E63, // Pink
            0xFF9C27B0, // Purple
            0xFF673AB7, // Deep Purple
            0xFF3F51B5, // Indigo
            0xFF2196F3, // Blue
            0xFF03A9F4, // Light Blue
            0xFF00BCD4, // Cyan
            0xFF009688, // Teal
            0xFF4CAF50, // Green
            0xFF8BC34A, // Light Green
            0xFFCDDC39, // Lime
            0xFFFFEB3B, // Yellow
            0xFFFFC107, // Amber
            0xFFFF9800, // Orange
            0xFFFF5722, // Deep Orange
            0xFF795548, // Brown
            0xFF9E9E9E, // Gray
            0xFF607D8B  // Blue Gray
        };

        int index = Math.abs(label % baseColors.length);
        int color = baseColors[index];
        return Color.argb(alpha, Color.red(color),
                         Color.green(color), Color.blue(color));
    }

    public FragmentRender(Context context, AttributeSet attrs) {
        super(context, attrs);

        textPaint.setColor(Color.WHITE);
        textPaint.setTypeface(Typeface.DEFAULT_BOLD);
        textPaint.setStyle(Paint.Style.FILL);
        textPaint.setTextSize(30);
    }

    public void setCoordsList(ArrayList<RectangleBox> newBoxList) {
        lock.lock();
        try {
            boxList.clear();
            boxList.addAll(newBoxList);
        } finally {
            lock.unlock();
        }
        postInvalidate();
    }

    public void render(Bitmap image, Size cameraSize, float fps,
                      long inferTime, long preprocessTime, long postprocessTime,
                      int displayRotation) {
        this.bitmap = image;
        this.cameraSize = cameraSize;
        this.fps = fps;
        this.inferTime = inferTime;
        this.preprocessTime = preprocessTime;
        this.postprocessTime = postprocessTime;
        this.displayRotation = displayRotation;
        postInvalidate();
    }

    @Override
    protected void onDraw(Canvas canvas) {
        lock.lock();

        if (bitmap == null || cameraSize == null) {
            lock.unlock();
            return;
        }

        try {
            // Calculate aspect-ratio-preserving dimensions
            float canvasRatio = (float) getWidth() / getHeight();
            float bitmapRatio = (float) bitmap.getWidth() / bitmap.getHeight();

            int insetWidth, insetHeight;
            if (canvasRatio > bitmapRatio) {
                insetHeight = getHeight();
                insetWidth = (int) (getHeight() * bitmapRatio);
            } else {
                insetWidth = getWidth();
                insetHeight = (int) (getWidth() / bitmapRatio);
            }

            int offsetWidth = (getWidth() - insetWidth) / 2;
            int offsetHeight = (getHeight() - insetHeight) / 2;

            // Setup transformation matrix for rotation
            transform.reset();
            float tx = getWidth() / 2.0f;
            float ty = getHeight() / 2.0f;

            switch (displayRotation) {
                case 0:
                    break;
                case 1:
                    transform.preRotate(-90, tx, ty);
                    break;
                case 3:
                    transform.preRotate(90, tx, ty);
                    break;
            }

            // Draw camera frame
            targetRect.set(offsetWidth, offsetHeight,
                          offsetWidth + insetWidth,
                          offsetHeight + insetHeight);

            canvas.save();
            canvas.concat(transform);
            canvas.drawBitmap(bitmap, null, targetRect, null);
            canvas.restore();

            // Draw bounding boxes
            for (RectangleBox box : boxList) {
                // Transform box coordinates
                float[] p0 = {box.left, box.top};
                float[] p1 = {box.right, box.bottom};
                transform.mapPoints(p0);
                transform.mapPoints(p1);

                float left = Math.min(p0[0], p1[0]);
                float upper = Math.min(p0[1], p1[1]);

                // Set color based on class with alpha based on confidence
                int alpha = (int) (255 * box.confidence);
                int color = labelColor(box.classIdx, alpha);

                // Draw bounding box
                framePaint.setColor(color);
                framePaint.setStyle(Paint.Style.STROKE);
                framePaint.setStrokeWidth(6);
                canvas.drawRect(p0[0], p0[1], p1[0], p1[1], framePaint);

                // Draw label background
                float textWidth = textPaint.measureText(box.label);
                float textHeight = textPaint.getFontMetrics().bottom -
                                  textPaint.getFontMetrics().top - 8.0f;
                float buf = 2.0f;

                labelFramePaint.setColor(color);
                labelFramePaint.setStyle(Paint.Style.FILL);
                canvas.drawRect(left, upper,
                              left + textWidth + 2 * buf,
                              upper - textHeight - 2 * buf,
                              labelFramePaint);

                // Draw label text
                int white = Color.argb(alpha, 255, 255, 255);
                textPaint.setColor(white);
                canvas.drawText(box.label, left + buf,
                              upper + textPaint.getFontMetrics().top + buf + 17.0f,
                              textPaint);
            }
        } finally {
            lock.unlock();
        }
    }
}
```

### RectangleBox Data Model

```java
public class RectangleBox {
    public float top;
    public float bottom;
    public float left;
    public float right;
    public int classIdx;
    public String label;
    public float confidence;

    public RectangleBox(float top, float bottom, float left, float right,
                       int classIdx, String label, float confidence) {
        this.top = top;
        this.bottom = bottom;
        this.left = left;
        this.right = right;
        this.classIdx = classIdx;
        this.label = label;
        this.confidence = confidence;
    }
}
```

---

## SemanticSegmentation - Real-time Scene Understanding

### FragmentRender for Segmentation

**File**: `apps/android/SemanticSegmentation/src/main/java/com/quicinc/semanticsegmentation/FragmentRender.java`

```java
public class FragmentRender extends View {
    private Bitmap segmentedBitmap = null;
    private Size cameraSize = null;
    private float fps;
    private long inferTime, preprocessTime, postprocessTime;
    private final Rect targetRect = new Rect();
    private final Paint textPaint = new Paint();

    public FragmentRender(Context context, AttributeSet attrs) {
        super(context, attrs);

        textPaint.setColor(Color.WHITE);
        textPaint.setTypeface(Typeface.DEFAULT_BOLD);
        textPaint.setStyle(Paint.Style.FILL);
        textPaint.setTextSize(50);
    }

    public void render(Bitmap segmented, Size cameraSize, float fps,
                      long inferTime, long preprocessTime, long postprocessTime) {
        this.segmentedBitmap = segmented;
        this.cameraSize = cameraSize;
        this.fps = fps;
        this.inferTime = inferTime;
        this.preprocessTime = preprocessTime;
        this.postprocessTime = postprocessTime;
        postInvalidate();
    }

    @Override
    protected void onDraw(Canvas canvas) {
        if (segmentedBitmap == null) {
            return;
        }

        // Calculate dimensions
        float canvasRatio = (float) getWidth() / getHeight();
        float bitmapRatio = (float) segmentedBitmap.getWidth() /
                           segmentedBitmap.getHeight();

        int insetWidth, insetHeight;
        if (canvasRatio > bitmapRatio) {
            insetHeight = getHeight();
            insetWidth = (int) (getHeight() * bitmapRatio);
        } else {
            insetWidth = getWidth();
            insetHeight = (int) (getWidth() / bitmapRatio);
        }

        int offsetWidth = (getWidth() - insetWidth) / 2;
        int offsetHeight = (getHeight() - insetHeight) / 2;

        targetRect.set(offsetWidth, offsetHeight,
                      offsetWidth + insetWidth,
                      offsetHeight + insetHeight);

        // Draw segmented image
        canvas.drawBitmap(segmentedBitmap, null, targetRect, null);

        // Rotate and draw performance metrics
        canvas.rotate(90, 0, 0);
        canvas.translate(offsetHeight, -insetWidth - offsetWidth);

        canvas.drawText(
            String.format("FPS: %.0f", fps),
            15, 50, textPaint
        );
        canvas.drawText(
            String.format("Preprocess: %.0f ms", preprocessTime / 1_000_000.0f),
            15, 55 + 60 * 2, textPaint
        );
        canvas.drawText(
            String.format("Infer: %.0f ms", inferTime / 1_000_000.0f),
            15, 55 + 60 * 3, textPaint
        );
        canvas.drawText(
            String.format("Postprocess: %.0f ms", postprocessTime / 1_000_000.0f),
            15, 55 + 60 * 4, textPaint
        );
        canvas.drawText(
            "Note: Will only produce sensible results on street scenes",
            15, insetWidth - 15, textPaint
        );
    }
}
```

---

## SuperResolution - Image Enhancement

The UI implementation is nearly identical to ImageClassification with these differences:

1. **Display Strategy**: Shows the enhanced/upscaled image after inference
2. **Input Handling**: Automatically downscales input images to model input size before upscaling

```java
private void loadImageFromAssets(String imageName) {
    backgroundExecutor.execute(() -> {
        InputStream stream = getAssets().open("images/" + imageName);
        Bitmap originalBitmap = BitmapFactory.decodeStream(stream);

        // Resize to model input size (will be upscaled by model)
        int[] inputSize = upscaler.getInputWidthHeight();
        Bitmap resizedBitmap = ImageProcessing.resizeAndPadMaintainAspectRatio(
            originalBitmap,
            inputSize[0],
            inputSize[1],
            0xFF
        );

        mainLooperHandler.post(() -> {
            selectedImage = resizedBitmap;
            selectedImageView.setImageBitmap(resizedBitmap);
            setInferenceUIEnabled(true);
        });
    });
}

private void runInferenceAsync() {
    setInferenceUIEnabled(false);

    backgroundExecutor.execute(() -> {
        // Generate upscaled image
        Bitmap upscaledImage = upscaler.generateUpscaledImage(selectedImage);

        long inferenceTime = upscaler.getLastInferenceTime();
        long totalTime = upscaler.getLastPreprocessingTime() +
                        inferenceTime +
                        upscaler.getLastPostprocessingTime();

        mainLooperHandler.post(() -> {
            // Display upscaled result
            selectedImageView.setImageBitmap(upscaledImage);
            inferenceTimeView.setText(inferenceTime + " ms");
            predictionTimeView.setText(totalTime + " ms");
            setInferenceUIEnabled(true);
        });
    });
}
```

---

## Threading Architecture

### Consistent Pattern Across All Apps

```java
public class BaseActivity extends AppCompatActivity {
    // Background executor for heavy operations
    protected ExecutorService backgroundExecutor;

    // Handler for UI updates on main thread
    protected Handler mainLooperHandler;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // Initialize threading
        backgroundExecutor = Executors.newSingleThreadExecutor();
        mainLooperHandler = new Handler(Looper.getMainLooper());
    }

    protected void runAsync(Runnable backgroundTask, Runnable uiTask) {
        backgroundExecutor.execute(() -> {
            backgroundTask.run();
            mainLooperHandler.post(uiTask);
        });
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        backgroundExecutor.shutdown();
    }
}
```

### Usage Pattern

```java
// Model loading
runAsync(
    () -> {
        // Background: Load model
        model = new ModelWrapper(this, modelPath);
    },
    () -> {
        // UI thread: Update UI
        setInferenceUIEnabled(true);
        Toast.makeText(this, "Model loaded", Toast.LENGTH_SHORT).show();
    }
);

// Inference
runAsync(
    () -> {
        // Background: Run inference
        result = model.predict(input);
        timing = model.getLastInferenceTime();
    },
    () -> {
        // UI thread: Display results
        resultView.setText(result);
        timingView.setText(timing + " ms");
    }
);
```

---

## Themes and Styling

### Color Schemes

**ChatApp** (`colors.xml`):
```xml
<resources>
    <color name="purple_200">#FFBB86FC</color>
    <color name="purple_500">#FF6200EE</color>
    <color name="purple_700">#FF3700B3</color>
    <color name="teal_200">#FF03DAC5</color>
    <color name="teal_700">#FF018786</color>
    <color name="black">#FF000000</color>
    <color name="white">#FFFFFFFF</color>
    <color name="user_color">#2196F3</color>
</resources>
```

**ImageClassification & SuperResolution** (`colors.xml`):
```xml
<resources>
    <color name="purple_qcom">#6D1683</color>
    <color name="purple_200">#FFBB86FC</color>
    <color name="purple_500">#FF6200EE</color>
    <color name="purple_700">#FF3700B3</color>
    <color name="teal_200">#FF03DAC5</color>
    <color name="teal_700">#FF018786</color>
    <color name="black">#FF000000</color>
    <color name="white">#FFFFFFFF</color>
</resources>
```

**ObjectDetection & SemanticSegmentation** (`colors.xml`):
```xml
<resources>
    <color name="purple_200">#FFBB86FC</color>
    <color name="purple_500">#FF6200EE</color>
    <color name="purple_700">#FF3700B3</color>
    <color name="teal_200">#FF03DAC5</color>
    <color name="teal_700">#FF018786</color>
    <color name="black">#FF000000</color>
    <color name="white">#FFFFFFFF</color>
    <color name="green">#4CAF50</color>
</resources>
```

### Theme Definitions

**ChatApp** (`themes.xml`):
```xml
<resources xmlns:tools="http://schemas.android.com/tools">
    <style name="Theme.ChatApp"
           parent="Theme.MaterialComponents.DayNight.DarkActionBar">
        <item name="colorPrimary">@color/purple_500</item>
        <item name="colorPrimaryVariant">@color/purple_700</item>
        <item name="colorOnPrimary">@color/white</item>
        <item name="colorSecondary">@color/teal_200</item>
        <item name="colorSecondaryVariant">@color/teal_700</item>
        <item name="colorOnSecondary">@color/black</item>
    </style>
</resources>
```

**Custom Spinner Theme** (ImageClassification):
```xml
<style name="spinnerTheme" parent="android:Widget.Spinner">
    <item name="android:textColorPrimary">@color/white</item>
</style>
```

### String Resources

**ChatApp** (`strings.xml`):
```xml
<resources>
    <string name="app_name">ChatApp</string>
    <string name="chat_with_llm">Chat with Llama 3.2 3B</string>
    <string name="user_hint_msg">What\'s on your mind?</string>
    <string name="sends_user_message">Sends user message</string>
</resources>
```

**ObjectDetection** (`strings.xml`):
```xml
<resources>
    <string name="app_name">ObjectDetection</string>
    <string name="request_permission">This app needs camera permission</string>
    <string name="camera_error">This device doesn\'t support Camera2 API.</string>
</resources>
```

---

## Best Practices

### 1. Always Use Background Threads for Heavy Operations

```java
// ❌ BAD: Running on UI thread
protected void onCreate(Bundle savedInstanceState) {
    model = new ModelWrapper(this); // Blocks UI!
}

// ✅ GOOD: Running on background thread
protected void onCreate(Bundle savedInstanceState) {
    ExecutorService executor = Executors.newSingleThreadExecutor();
    executor.execute(() -> {
        model = new ModelWrapper(this);
        runOnUiThread(() -> {
            // Update UI
        });
    });
}
```

### 2. Provide Visual Feedback

```java
// Show loading state
progressBar.setVisibility(View.VISIBLE);
button.setEnabled(false);
resultText.setText("Processing...");

// Run operation
performOperation();

// Hide loading state
progressBar.setVisibility(View.GONE);
button.setEnabled(true);
resultText.setText(result);
```

### 3. Handle Errors Gracefully

```java
backgroundExecutor.execute(() -> {
    try {
        result = model.predict(input);
        mainLooperHandler.post(() -> {
            displayResult(result);
        });
    } catch (Exception e) {
        mainLooperHandler.post(() -> {
            Toast.makeText(this, "Error: " + e.getMessage(),
                         Toast.LENGTH_LONG).show();
            Log.e(TAG, "Inference error", e);
        });
    }
});
```

### 4. Clean Up Resources

```java
@Override
protected void onDestroy() {
    super.onDestroy();

    // Close model
    if (model != null) {
        model.close();
    }

    // Shutdown executor
    if (backgroundExecutor != null) {
        backgroundExecutor.shutdown();
    }

    // Close camera
    if (cameraDevice != null) {
        cameraDevice.close();
    }
}
```

### 5. Use Appropriate Layouts

```java
// ✅ GOOD: ConstraintLayout for complex UIs
<ConstraintLayout>
    <TextView app:layout_constraintTop_toTopOf="parent" />
    <Button app:layout_constraintTop_toBottomOf="@id/textView" />
</ConstraintLayout>

// ✅ GOOD: RecyclerView for lists
<RecyclerView
    android:id="@+id/recyclerView"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />

// ❌ BAD: Nested LinearLayouts
<LinearLayout>
    <LinearLayout>
        <LinearLayout>
            <!-- Too many nested layouts -->
        </LinearLayout>
    </LinearLayout>
</LinearLayout>
```

### 6. Implement Proper Permission Handling

```java
private void checkCameraPermission() {
    if (ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA)
        != PackageManager.PERMISSION_GRANTED) {

        // Request permission
        ActivityCompat.requestPermissions(
            this,
            new String[]{Manifest.permission.CAMERA},
            CAMERA_PERMISSION_CODE
        );
    } else {
        // Permission already granted
        startCamera();
    }
}

@Override
public void onRequestPermissionsResult(int requestCode, String[] permissions,
                                      int[] grantResults) {
    if (requestCode == CAMERA_PERMISSION_CODE) {
        if (grantResults.length > 0 &&
            grantResults[0] == PackageManager.PERMISSION_GRANTED) {
            startCamera();
        } else {
            Toast.makeText(this, "Camera permission required",
                         Toast.LENGTH_LONG).show();
        }
    }
}
```

### 7. Use Material Design Components

```xml
<!-- CardView for elevation and rounded corners -->
<androidx.cardview.widget.CardView
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    app:cardElevation="4dp"
    app:cardCornerRadius="8dp">

    <ImageView
        android:layout_width="match_parent"
        android:layout_height="match_parent" />
</androidx.cardview.widget.CardView>

<!-- Material Button with ripple effect -->
<com.google.android.material.button.MaterialButton
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="Run Model"
    app:cornerRadius="8dp" />
```

### 8. Optimize RecyclerView Performance

```java
// Use ViewHolder pattern
public class MyAdapter extends RecyclerView.Adapter<MyAdapter.ViewHolder> {

    static class ViewHolder extends RecyclerView.ViewHolder {
        TextView textView;
        ImageView imageView;

        ViewHolder(View view) {
            super(view);
            textView = view.findViewById(R.id.text);
            imageView = view.findViewById(R.id.image);
        }
    }

    @Override
    public void onBindViewHolder(ViewHolder holder, int position) {
        // Bind data to views
        holder.textView.setText(data.get(position));
    }
}

// Set fixed size if known
recyclerView.setHasFixedSize(true);

// Use appropriate layout manager
recyclerView.setLayoutManager(new LinearLayoutManager(this));
```

### 9. Handle Configuration Changes

```java
// Retain data across configuration changes
@Override
public void onSaveInstanceState(Bundle outState) {
    super.onSaveInstanceState(outState);
    outState.putString("result", currentResult);
    outState.putLong("inferenceTime", lastInferenceTime);
}

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    if (savedInstanceState != null) {
        currentResult = savedInstanceState.getString("result");
        lastInferenceTime = savedInstanceState.getLong("inferenceTime");

        // Restore UI state
        resultView.setText(currentResult);
        timingView.setText(lastInferenceTime + " ms");
    }
}
```

### 10. Use Appropriate Image Loading

```java
// Load from assets
InputStream stream = getAssets().open("images/sample.png");
Bitmap bitmap = BitmapFactory.decodeStream(stream);

// Load from URI (gallery)
InputStream stream = getContentResolver().openInputStream(uri);
Bitmap bitmap = BitmapFactory.decodeStream(stream);

// Always close streams
try (InputStream stream = getAssets().open("image.png")) {
    Bitmap bitmap = BitmapFactory.decodeStream(stream);
} catch (IOException e) {
    Log.e(TAG, "Error loading image", e);
}
```

---

## Summary

This guide covers all UI implementation patterns across the Qualcomm AI Hub Android sample apps:

1. **ChatApp**: Conversational interface with RecyclerView, streaming responses, and chat bubbles
2. **ImageClassification**: Static image analysis with Spinner, CardView, and delegate selection
3. **ObjectDetection**: Real-time camera with Camera2 API and custom bounding box rendering
4. **SemanticSegmentation**: Real-time segmentation with custom overlay rendering
5. **SuperResolution**: Image enhancement with before/after display

### Key Takeaways

- **Consistent threading**: All apps use ExecutorService + Handler pattern
- **Material Design**: CardView, ConstraintLayout, Material buttons
- **Custom rendering**: Canvas-based custom views for overlays
- **Performance metrics**: All apps display timing information
- **Delegate selection**: Hardware acceleration options (CPU/GPU/NPU)
- **Proper resource management**: Always clean up in onDestroy()

### File Locations Reference

- ChatApp layouts: `apps/android/ChatApp/src/main/res/layout/`
- ImageClassification layouts: `apps/android/ImageClassification/src/main/res/layout/`
- ObjectDetection custom view: `apps/android/ObjectDetection/src/main/java/com/quicinc/objectdetection/FragmentRender.java`
- Camera fragment: `apps/android/ObjectDetection/src/main/java/com/quicinc/objectdetection/CameraFragment.java`
- Conversation activity: `apps/android/ChatApp/src/main/java/com/quicinc/chatapp/Conversation.java`

---

## License

This documentation is based on sample code from Qualcomm AI Hub Android Sample Apps.
Copyright (c) 2025 Qualcomm Technologies, Inc. and/or its subsidiaries.
SPDX-License-Identifier: BSD-3-Clause
