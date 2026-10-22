# 100 Days of AI Engineering

# Module 1: AI Foundations

## Days 1-6

### Day 1: What Is Artificial Intelligence?

**Tutorial:** [Day 1: What Is Artificial Intelligence?](100-Day%20Tracker/Day-01-What-Is-Artificial-Intelligence.md)

Study:

-   Artificial intelligence
-   Machine learning
-   Deep learning
-   Generative AI
-   Narrow AI
-   General intelligence as a concept
-   Symbolic vs statistical AI
-   Predictive vs generative systems
-   Training vs inference
-   Models, parameters and algorithms
-   AI engineering vs ML engineering vs data science vs research

**Output:** Create an AI field map showing how ML, DL, CV, NLP, LLMs,
agents and robotics AI relate.

### Day 2: How Machines Learn

**Tutorial:** [Day 2: How Machines Learn](100-Day%20Tracker/Day-02-How-Machines-Learn.md)

Study:

-   Data
-   Features
-   Labels
-   Models
-   Parameters
-   Training
-   Loss
-   Optimization
-   Generalization
-   Inference
-   Supervised learning
-   Unsupervised learning
-   Self-supervised learning
-   Reinforcement learning

### Day 3: The AI/ML Lifecycle

**Tutorial:** [Day 3: The AI/ML Lifecycle](100-Day%20Tracker/Day-03-The-AI-ML-Lifecycle.md)

Study:

Problem definition → data collection → exploration → cleaning → features
→ baseline → training → validation → testing → evaluation → deployment →
monitoring → iteration.

Learn why AI engineering is mostly the work around `model.fit()`.

### Day 4: AI Problem Formulation

**Tutorial:** [Day 4: AI Problem Formulation](100-Day%20Tracker/Day-04-AI-Problem-Formulation.md)

Study:

-   Classification
-   Regression
-   Ranking
-   Clustering
-   Generation
-   Detection
-   Segmentation
-   Forecasting
-   Recommendation
-   Anomaly detection
-   Control/decision-making

**Exercise:** Take 20 real problems and identify the correct ML
formulation.

### Day 5: AI Development Environment

**Tutorial:** [Day 5: AI Development Environment](100-Day%20Tracker/Day-05-AI-Development-Environment.md)

Learn:

-   Python AI ecosystem
-   Jupyter notebooks
-   Virtual environments
-   NumPy
-   pandas
-   Matplotlib
-   scikit-learn
-   PyTorch
-   Hugging Face ecosystem
-   CPU vs GPU
-   CUDA concept
-   Reproducibility
-   Random seeds

### Day 6: First End-to-End AI Experiment

Take a small dataset through:

-   loading
-   inspection
-   preprocessing
-   train/test split
-   baseline model
-   metrics
-   predictions
-   error inspection
-   documentation

------------------------------------------------------------------------

# Module 2: Mathematics for AI

## Days 7-16

### Day 7: Mathematical Foundations

Review:

-   Functions
-   Variables
-   Equations
-   Exponents
-   Logarithms
-   Summation notation
-   Function composition
-   Graphs

### Day 8: Vectors

Study:

-   Scalars
-   Vectors
-   Vector addition
-   Scalar multiplication
-   Magnitude
-   Unit vectors
-   Dot products
-   Geometric interpretation

Implement operations with NumPy.

### Day 9: Matrices

Study:

-   Matrices
-   Shapes
-   Matrix addition
-   Multiplication
-   Transpose
-   Identity
-   Inverse concept
-   Systems of equations

Connect matrices to datasets and neural-network layers.

### Day 10: Linear Algebra for ML

Study:

-   Linear transformations
-   Basis
-   Rank
-   Orthogonality
-   Eigenvalues/eigenvectors
-   Singular Value Decomposition concept
-   Dimensionality reduction intuition

### Day 11: Calculus I

Study:

-   Limits concept
-   Derivatives
-   Slopes
-   Partial derivatives
-   Chain rule
-   Gradients

Connect derivatives to learning.

### Day 12: Calculus II: Optimization

Study:

-   Objective functions
-   Minima/maxima
-   Gradient descent
-   Learning rate
-   Convexity intuition
-   Local/global minima
-   Stochastic optimization

Implement gradient descent manually.

### Day 13: Probability

Study:

-   Events
-   Random variables
-   Conditional probability
-   Independence
-   Bayes' theorem
-   Probability distributions

### Day 14: Statistics

Study:

-   Mean
-   Median
-   Variance
-   Standard deviation
-   Covariance
-   Correlation
-   Sampling
-   Bias
-   Confidence concepts
-   Statistical distributions

### Day 15: Information Theory

Study:

-   Information
-   Entropy
-   Cross-entropy
-   KL divergence concept
-   Why cross-entropy appears in classification and language models

### Day 16: Mathematics Project

Implement from scratch:

-   vectors/matrices
-   linear regression
-   loss calculation
-   gradient descent

Avoid ML frameworks for this project.

------------------------------------------------------------------------

# Module 3: Data Engineering for AI

## Days 17-23

### Day 17: NumPy

Study:

-   ndarrays
-   shapes
-   dimensions
-   indexing/slicing
-   broadcasting
-   vectorization
-   matrix operations
-   random generation

### Day 18: pandas

Study:

-   Series
-   DataFrames
-   indexing
-   filtering
-   grouping
-   merging
-   missing data
-   transformations

### Day 19: Data Cleaning

Study:

-   Missing values
-   Duplicates
-   Invalid records
-   Outliers
-   Data types
-   Leakage
-   Label errors
-   Data quality

### Day 20: Exploratory Data Analysis

Study:

-   Distributions
-   Relationships
-   Correlations
-   Class imbalance
-   Visualization
-   Hypothesis formation
-   Data-driven debugging

### Day 21: Feature Engineering

Study:

-   Numerical features
-   Categorical encoding
-   Scaling
-   Normalization
-   Transformations
-   Feature interactions
-   Feature selection

### Day 22: Datasets and Data Pipelines

Study:

-   Dataset splits
-   Data loaders
-   Batch processing
-   Shuffling
-   Dataset versioning concepts
-   Reproducible preprocessing
-   Training/serving consistency

### Day 23: Data Project

Take a messy dataset and produce:

-   cleaned dataset
-   EDA report
-   preprocessing pipeline
-   train/validation/test strategy
-   documented assumptions

------------------------------------------------------------------------

# Module 4: Classical Machine Learning

## Days 24-38

### Day 24: ML Workflow and Baselines

Study:

-   Baseline models
-   Train/validation/test
-   Generalization
-   Underfitting
-   Overfitting
-   Bias/variance
-   Cross-validation

### Day 25: Linear Regression

Study and derive:

-   Linear model
-   Residuals
-   MSE
-   Least-squares intuition
-   Gradient descent
-   Regularization introduction

Implement from scratch, then compare with scikit-learn.

### Day 26: Logistic Regression

Study:

-   Classification
-   Logits
-   Sigmoid
-   Probabilities
-   Decision boundaries
-   Log loss
-   Thresholds

### Day 27: k-Nearest Neighbors

Study:

-   Distance metrics
-   Neighborhoods
-   Choosing k
-   Scaling sensitivity
-   Classification/regression

### Day 28: Naive Bayes

Study:

-   Bayes' theorem
-   Conditional probability
-   Independence assumption
-   Text-classification applications

### Day 29: Decision Trees

Study:

-   Splits
-   Nodes/leaves
-   Impurity
-   Gini
-   Entropy
-   Information gain
-   Overfitting
-   Tree depth

### Day 30: Ensemble Learning

Study:

-   Bagging
-   Random forests
-   Boosting
-   Gradient boosting
-   Voting
-   Stacking concepts

### Day 31: Support Vector Machines

Study:

-   Hyperplanes
-   Margins
-   Support vectors
-   Kernels concept
-   Linear vs nonlinear separation

### Day 32: Clustering

Study:

-   Unsupervised learning
-   k-means
-   Centroids
-   Distance
-   Initialization
-   Hierarchical clustering concept
-   DBSCAN concept

### Day 33: Dimensionality Reduction

Study:

-   Curse of dimensionality
-   PCA
-   Variance
-   Projection
-   t-SNE/UMAP concepts and limitations

### Day 34: Model Evaluation: Classification

Study:

-   Confusion matrix
-   Accuracy
-   Precision
-   Recall
-   F1
-   Specificity concept
-   ROC/AUC
-   PR curves
-   Threshold selection

### Day 35: Model Evaluation: Regression

Study:

-   MAE
-   MSE
-   RMSE
-   R²
-   Residual analysis

### Day 36: Hyperparameters and Model Selection

Study:

-   Parameters vs hyperparameters
-   Grid search
-   Random search
-   Cross-validation
-   Leakage
-   Pipeline-based evaluation

### Day 37: Interpretability and Error Analysis

Study:

-   Feature importance
-   Model coefficients
-   Partial dependence concept
-   SHAP concept
-   Error slices
-   Failure analysis
-   Dataset problems vs model problems

### Day 38: Classical ML Project

Build a complete ML product:

-   data pipeline
-   baseline
-   several models
-   experiment comparison
-   evaluation
-   error analysis
-   model selection
-   simple inference interface

------------------------------------------------------------------------

# Module 5: Deep Learning Foundations

## Days 39-51

### Day 39: Neural Networks

Study:

-   Biological inspiration vs actual computation
-   Artificial neurons
-   Inputs
-   Weights
-   Biases
-   Layers
-   Activations
-   Forward pass

### Day 40: Perceptrons and MLPs

Study:

-   Perceptron
-   Linear separability
-   Multi-layer perceptrons
-   Hidden layers
-   Representation learning

### Day 41: Activation Functions

Study:

-   Sigmoid
-   Tanh
-   ReLU
-   Leaky ReLU
-   GELU concept
-   Softmax
-   Why nonlinearities matter

### Day 42: Loss Functions

Study:

-   MSE
-   Binary cross-entropy
-   Categorical cross-entropy
-   Negative log likelihood
-   Task/loss alignment

### Day 43: Backpropagation

Go deeply into:

-   Computational graphs
-   Chain rule
-   Forward pass
-   Loss
-   Gradients
-   Backward pass
-   Parameter updates

Implement a tiny neural network manually.

### Day 44: Optimization

Study:

-   Batch gradient descent
-   SGD
-   Mini-batches
-   Momentum
-   Adam
-   Learning-rate schedules
-   Optimization instability

### Day 45: Generalization and Regularization

Study:

-   Overfitting
-   Weight decay
-   Dropout
-   Early stopping
-   Data augmentation
-   Batch normalization concept

### Day 46: PyTorch Tensors

Study:

-   Tensors
-   Shapes
-   Devices
-   Dtypes
-   Tensor operations
-   CPU/GPU movement

### Day 47: Autograd

Study:

-   Computational graphs
-   `requires_grad`
-   Gradient accumulation
-   `backward`
-   Detaching
-   Gradient control

### Day 48: Building PyTorch Models

Study:

-   `nn.Module`
-   Layers
-   Forward methods
-   Losses
-   Optimizers
-   Model architecture

### Day 49: Datasets and DataLoaders

Study:

-   Dataset abstraction
-   Batches
-   Shuffling
-   Transforms
-   DataLoader
-   Custom datasets

### Day 50: Training Loops

Build a complete loop:

-   train mode
-   forward
-   loss
-   zero gradients
-   backward
-   optimizer step
-   validation
-   checkpointing
-   metrics

### Day 51: Deep Learning Project

Train, evaluate and document a neural network from scratch using
PyTorch.

------------------------------------------------------------------------

# Module 6: Computer Vision

## Days 52-61

### Day 52: Digital Images

Study:

-   Pixels
-   Channels
-   RGB/grayscale
-   Resolution
-   Image arrays
-   Color spaces
-   Normalization

### Day 53: Classical Image Processing

Study:

-   Resizing
-   Cropping
-   Thresholding
-   Blurring
-   Edges
-   Morphological operations concepts
-   Contours/features concepts

### Day 54: Convolution

Study:

-   Kernels
-   Filters
-   Stride
-   Padding
-   Feature maps
-   Receptive fields

Implement simple convolution manually.

### Day 55: Convolutional Neural Networks

Study:

-   Convolution layers
-   Pooling
-   Feature hierarchies
-   Classifiers
-   CNN architecture

### Day 56: Image Classification

Build: dataset → augmentation → CNN → training → metrics → inference.

### Day 57: Transfer Learning

Study:

-   Pretrained models
-   Feature extraction
-   Fine-tuning
-   Frozen layers
-   Domain adaptation concepts

### Day 58: Object Detection

Study:

-   Bounding boxes
-   IoU
-   Confidence
-   NMS
-   One-stage/two-stage detector concepts

### Day 59: Image Segmentation

Study:

-   Semantic segmentation
-   Instance segmentation
-   Masks
-   Pixel-level prediction

### Day 60: Vision for Robotics and Edge Systems

Study:

-   Real-time inference
-   Latency
-   FPS
-   Camera pipelines
-   Detection-to-action pipelines
-   Compute limitations

### Day 61: Computer Vision Project

Create a vision application that trains/evaluates a model and performs
inference on new visual data.

------------------------------------------------------------------------

# Module 7: NLP and Sequence Modeling

## Days 62-69

### Day 62: NLP Foundations

Study:

-   Text as data
-   Corpora
-   Tokens
-   Vocabulary
-   Normalization
-   Tokenization
-   NLP tasks

### Day 63: Text Representations

Study:

-   Bag-of-words
-   n-grams
-   TF-IDF
-   Sparse vectors
-   Semantic limitations

### Day 64: Embeddings

Study:

-   Dense representations
-   Word embeddings
-   Similarity
-   Cosine similarity
-   Semantic spaces
-   Contextual embeddings introduction

### Day 65: Sequence Models

Study:

-   Sequence dependence
-   RNNs
-   Hidden states
-   Vanishing gradients
-   LSTM/GRU concepts

### Day 66: Attention

Study:

-   Query
-   Key
-   Value
-   Attention scores
-   Weighted representations
-   Self-attention intuition

### Day 67: Transformers

Study:

-   Token embeddings
-   Positional information
-   Multi-head attention
-   Feed-forward layers
-   Residual connections
-   Layer normalization
-   Encoder vs decoder

### Day 68: NLP with Transformers

Explore:

-   Classification
-   Named-entity recognition
-   Summarization
-   Translation
-   Question answering

### Day 69: NLP Project

Build and evaluate a transformer-based NLP application.

------------------------------------------------------------------------

# Module 8: Large Language Models

## Days 70-80

### Day 70: What Is an LLM?

Study:

-   Language modeling
-   Next-token prediction
-   Tokens
-   Context
-   Parameters
-   Pretraining
-   Inference
-   Foundation models

### Day 71: Tokenization

Study:

-   Tokenizers
-   Subwords
-   Vocabulary
-   Special tokens
-   Context windows
-   Token counts
-   Why tokenization affects behavior

### Day 72: How LLMs Generate Text

Study:

-   Logits
-   Probabilities
-   Sampling
-   Greedy decoding
-   Temperature
-   Top-k/top-p concepts
-   Stopping criteria

### Day 73: Pretraining

Study:

-   Large corpora
-   Self-supervised learning
-   Training objective
-   Scaling concepts
-   Compute/data/model relationships

### Day 74: Fine-Tuning

Study:

-   Supervised fine-tuning
-   Instruction tuning
-   Full fine-tuning
-   Parameter-efficient fine-tuning
-   LoRA concepts
-   Dataset design

### Day 75: Alignment and Preference Optimization

Conceptually study:

-   Human preferences
-   Reward modeling
-   RLHF
-   DPO-style preference optimization
-   Safety/alignment goals and limitations

### Day 76: Prompt Engineering as Interface Design

Study:

-   Instructions
-   Context
-   Examples
-   Structured outputs
-   Constraints
-   Prompt templates
-   Prompt injection awareness
-   Reliability limitations

### Day 77: Embeddings and Semantic Search

Study:

-   Embedding models
-   Vector similarity
-   Semantic retrieval
-   Chunk representations
-   Vector indexes/databases concepts

### Day 78: Retrieval-Augmented Generation

Study the complete pipeline:

documents → parsing → chunking → embeddings → indexing → retrieval →
context construction → generation → citations/evaluation.

### Day 79: LLM Evaluation

Study:

-   Task success
-   Groundedness
-   Relevance
-   Hallucination/factuality concepts
-   Human evaluation
-   Automated evaluation
-   Regression test sets
-   Cost/latency/quality tradeoffs

### Day 80: LLM Application Project

Build a grounded knowledge assistant with retrieval, evaluation and
documented failure cases.

------------------------------------------------------------------------

# Module 9: AI Agents and Automation

## Days 81-88

### Day 81: What Is an AI Agent?

Study:

-   Model
-   Instructions
-   Tools
-   Environment
-   State
-   Actions
-   Observations
-   Goals
-   Agent loop

### Day 82: Tool Use and Function Calling

Study:

-   Tool schemas
-   Arguments
-   Structured outputs
-   Tool selection
-   Validation
-   Errors
-   Permissions
-   Tool-result handling

### Day 83: Agent State and Memory

Study:

-   Conversation state
-   Short-term context
-   External memory
-   Retrieval
-   State stores
-   Memory quality
-   When not to remember information

### Day 84: Planning and Workflows

Study:

-   Deterministic workflows
-   Agentic decision points
-   Task decomposition
-   Routing
-   Sequential/parallel workflows
-   Human approval checkpoints

### Day 85: Agent Frameworks

Learn how modern frameworks implement these ideas:

-   LangGraph
-   LlamaIndex
-   smolagents

Learn the architecture each library implements, because the
library itself is replaceable.

### Day 86: Agentic RAG

Combine:

-   Retrieval
-   Tool use
-   Query rewriting
-   Multi-step information gathering
-   Evidence
-   Generation
-   Evaluation

### Day 87: Agent Evaluation and Observability

Study:

-   Traces
-   Tool-call success
-   Task completion
-   Failure categories
-   Latency
-   Cost
-   Reliability
-   Regression tests
-   Human evaluation

### Day 88: Agent Project

Build an agent that safely uses multiple tools to complete a bounded
real-world workflow, with evaluation and logs.

------------------------------------------------------------------------

# Module 10: MLOps and LLMOps

## Days 89-95

### Day 89: Reproducible ML

Study:

-   Code versions
-   Dataset versions
-   Configurations
-   Seeds
-   Environments
-   Model artifacts
-   Reproducible experiments

### Day 90: Experiment Tracking

Study:

-   Runs
-   Parameters
-   Metrics
-   Artifacts
-   Model comparison
-   Experiment dashboards

Use MLflow or equivalent tooling.

### Day 91: Model Registry and Versioning

Study:

-   Model artifacts
-   Versions
-   Metadata
-   Promotion concepts
-   Reproducible loading
-   Rollback concepts

### Day 92: Model Serving

Study:

-   Batch vs online inference
-   Inference APIs
-   Model servers
-   Serialization
-   Containers
-   Latency
-   Throughput
-   Scaling concepts

### Day 93: Monitoring

Study:

-   Service health
-   Latency/errors
-   Data drift concept
-   Model performance
-   Feedback
-   LLM traces
-   Quality monitoring
-   Cost monitoring

### Day 94: AI Deployment Architecture

Study:

-   APIs
-   Queues
-   Databases
-   Object storage
-   GPUs
-   Containers
-   Cloud deployment
-   Caching
-   Scaling
-   Security and secrets

### Day 95: Production AI Project

Take a trained model or LLM application through:

-   experiment tracking
-   versioning
-   API serving
-   containerization
-   deployment
-   monitoring
-   documentation

------------------------------------------------------------------------

# Module 11: Edge AI and Robotics AI

## Days 96-98

### Day 96: Edge AI

Study:

-   Cloud vs edge inference
-   Latency
-   Bandwidth
-   Privacy
-   Power
-   Memory
-   Model size
-   Quantization concepts
-   Pruning concepts
-   Hardware acceleration

### Day 97: AI for Robotics

Study how AI connects to:

-   Perception
-   Detection
-   Segmentation
-   Pose estimation
-   Localization
-   Planning
-   Policies
-   Sensor data
-   Robot actions

Understand the boundary between classical robotics algorithms and
learned systems.

### Day 98: Reinforcement Learning and Embodied AI

Study:

-   Agent
-   Environment
-   State/observation
-   Action
-   Reward
-   Policy
-   Episode
-   Exploration/exploitation
-   Value concepts
-   Policy learning concepts
-   Simulation
-   Sim-to-real
-   Embodied/physical AI

------------------------------------------------------------------------

# Module 12: AI Engineering Capstone

## Days 99-100

### Day 99: Design the AI System

Start with a problem, not a model.

Produce:

1.  Problem statement
2.  Users
3.  Success criteria
4.  AI task formulation
5.  Data requirements
6.  Baseline
7.  Model strategy
8.  Evaluation plan
9.  Architecture
10. Training/inference pipeline
11. API/interface
12. Storage
13. Experiment tracking
14. Deployment strategy
15. Monitoring
16. Cost/latency requirements
17. Failure modes
18. Safety/privacy considerations
19. Test plan
20. Build backlog

### Day 100: Build, Evaluate, Deploy and Teach

Your capstone should demonstrate several layers:

**Data → Model → Evaluation → Application → Deployment → Monitoring**

Your final presentation should explain:

-   Problem
-   Dataset
-   Mathematics
-   Model architecture
-   Training
-   Experiments
-   Metrics
-   Error analysis
-   Application architecture
-   Deployment
-   Monitoring
-   Limitations
-   Tradeoffs
-   Future improvements

Suggested episode:

**Day 100/100: I Built and Deployed a Complete AI System From Scratch
\| 100 Days of AI Engineering**

------------------------------------------------------------------------

# Primary Learning Resources

## Mathematics / Machine Learning

-   scikit-learn User Guide:
    https://scikit-learn.org/stable/user_guide.html
-   NumPy: https://numpy.org/doc/
-   pandas: https://pandas.pydata.org/docs/

## Deep Learning

-   PyTorch Tutorials: https://docs.pytorch.org/tutorials/
-   PyTorch Learn the Basics:
    https://docs.pytorch.org/tutorials/beginner/basics/intro.html

## NLP / LLMs

-   Hugging Face Learn: https://huggingface.co/learn
-   Hugging Face LLM Course: https://huggingface.co/learn/llm-course/

## AI Agents

-   Hugging Face Agents Course:
    https://huggingface.co/learn/agents-course/
-   LangGraph documentation:
    https://docs.langchain.com/oss/python/langgraph/overview
-   LlamaIndex documentation: https://docs.llamaindex.ai/

## MLOps / LLMOps

-   MLflow Documentation: https://mlflow.org/docs/latest/
-   MLflow Tracking: https://mlflow.org/docs/latest/ml/tracking/
-   MLflow Deployment: https://mlflow.org/docs/latest/deployment/

## Edge / Robotics AI

-   NVIDIA Jetson AI resources:
    https://developer.nvidia.com/embedded/learn
-   NVIDIA Isaac Sim: https://developer.nvidia.com/isaac/sim

------------------------------------------------------------------------

# Recommended Documentation and Video Learning

## Core documentation/courses

-   Google Machine Learning Crash Course:
    https://developers.google.com/machine-learning/crash-course
-   NumPy: https://numpy.org/doc/
-   pandas: https://pandas.pydata.org/docs/
-   scikit-learn User Guide:
    https://scikit-learn.org/stable/user_guide.html
-   PyTorch Tutorials: https://docs.pytorch.org/tutorials/
-   Hugging Face Learn: https://huggingface.co/learn
-   Hugging Face LLM Course: https://huggingface.co/learn/llm-course/
-   Hugging Face Agents Course:
    https://huggingface.co/learn/agents-course/
-   MLflow: https://mlflow.org/docs/latest/
-   NVIDIA Jetson learning: https://developer.nvidia.com/embedded/learn

## YouTube / video sources

-   **3Blue1Brown:** linear algebra, calculus and neural-network
    intuition: https://www.youtube.com/@3blue1brown
-   **StatQuest with Josh Starmer:** statistics, classical ML, neural
    networks and PyTorch intuition: https://www.youtube.com/@statquest
-   **PyTorch official tutorials/video series:** use alongside the
    current PyTorch tutorial sequence:
    https://docs.pytorch.org/tutorials/
-   **Hugging Face:** transformers, open models and agent education:
    https://www.youtube.com/@HuggingFace
-   **MIT OpenCourseWare:** probability, linear algebra, algorithms
    and ML-related university material: https://www.youtube.com/@mitocw
-   **DeepLearningAI:** supplementary ML/deep-learning/LLM
    engineering explanations: https://www.youtube.com/@Deeplearningai

### Suggested viewing by phase

-   Math: 3Blue1Brown for intuition, then solve exercises yourself.
-   Classical ML: Google ML Crash Course + StatQuest + scikit-learn
    docs.
-   Deep learning: PyTorch official learning path + StatQuest for
    intuition.
-   Transformers/LLMs: Hugging Face course and videos.
-   Agents: Hugging Face Agents Course; focus on tools, workflows,
    evaluation and observability rather than framework memorization.
-   MLOps: MLflow docs and production case studies.

# Day 101: Career, Hiring and Monetization

## 101.1 Choose a Marketable Direction

-   Machine-learning engineer
-   Applied AI engineer
-   AI/LLM application engineer
-   Data/ML-oriented software engineer
-   Computer-vision engineer
-   NLP/LLM engineer
-   MLOps/LLMOps engineer after infrastructure depth
-   AI automation/agent engineer
-   Edge-AI engineer after embedded/robotics depth

## 101.2 What "Job Ready" Means

Show evidence that you can:

-   formulate an ML/AI problem correctly
-   prepare and analyze data
-   establish baselines
-   train models
-   choose meaningful metrics
-   perform error analysis
-   build an inference/application layer
-   evaluate LLM/agent systems rather than judging demos by feel
-   deploy and monitor an AI system
-   explain limitations, data issues, latency/cost and failure modes

A stronger proof package is:

1.  **2-4 serious projects** relevant to the role.
2.  Clean source repositories with useful READMEs.
3.  A short technical case study for each major project.
4.  A live demo, video demo, simulation, or deployed application where
    appropriate.
5.  Tests, measurements, evaluation results, or engineering evidence.
6.  Clear explanation of tradeoffs and failures.
7.  Consistent Git history showing that you actually built the work.
8.  The ability to explain your own code/design without relying on
    notes.

## 101.3 Build a Proof-of-Work Portfolio

Every flagship project should answer:

-   What problem did I solve?
-   Who is it for?
-   What constraints did I have?
-   Why did I choose this architecture?
-   What did I personally implement?
-   What failed?
-   How did I debug it?
-   How did I test/evaluate it?
-   What are the measurable results?
-   What would I change in version 2?

Publish a portfolio page containing a concise bio, skills, projects,
technical write-ups, Git repositories, demos and a contact method.

## 101.4 Getting Hired

### Resume, applications, networking, interviews

Prepare for:

-   explaining projects deeply
-   debugging
-   fundamentals
-   practical coding/design tasks
-   architecture/tradeoff discussions
-   behavioral questions about teamwork and failure
-   Python, SQL and software fundamentals
-   probability/statistics and ML fundamentals
-   model evaluation and leakage
-   deep-learning fundamentals for DL roles
-   practical ML/LLM system design for applied roles

## 101.5 Ways to Make Money

A technical skill can produce income in several ways:

-   Employment and internships
-   Contract projects
-   Productized services
-   Products
-   Education and technical content
-   Consulting, later

## 101.6 The First-Income Ladder

**Stage 1, Proof:** Complete small projects and publish them.

**Stage 2, Reliability:** Complete larger projects with
documentation, tests and measurable results.

**Stage 3, Small paid problem:** Solve a bounded problem for a real
person, school, community, organization or business where legally and
practically appropriate.

**Stage 4, Repeatability:** Turn repeated work into a reusable
service/product.

**Stage 5, Specialization:** Become known for solving a narrower,
valuable problem extremely well.

**Stage 6, Scale:** Employment growth, higher-value contracts,
products, content, or eventually a company.

## 101.7 Your Career Capstone

Create a **Career Launch Pack**:

-   one-page role target
-   skills-gap checklist
-   resume
-   portfolio website
-   3 flagship case studies
-   GitHub/repository cleanup
-   90-second introduction
-   project demo reel
-   20 practice interview questions
-   list of target organizations/roles
-   application tracker
-   90-day continued-learning plan
-   one realistic monetization experiment
