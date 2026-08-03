---
author: "Nishant M Gandhi"
date: 2024-07-15
linktitle: "Composition Over Inheritance: Building a Single ML Training Interface"
title: "Composition Over Inheritance: Building a Single ML Training Interface"
image: ""
---

## The Problem

We needed to build a Python training library that supported regression, classification, sentiment analysis, and speech-to-text models through a single unified interface. The same `train()`, `predict()`, `compute_metrics()` API for all of them.

Inheritance seemed natural. Create a `BaseModel` abstract class. Each model type inherits and overrides the methods it needs. Done.

This doesn't work for ML systems. The problems show up immediately.

## Why Inheritance Fails for ML

### Rigid Contracts

```python
# Inheritance approach
class BaseModel(ABC):
    @abstractmethod
    def validate_input(self, data: Dict) -> bool:
        pass
    
    @abstractmethod
    def preprocess(self, data: Dict) -> np.ndarray:
        pass
    
    @abstractmethod
    def train(self, X: np.ndarray, y: np.ndarray) -> None:
        pass
    
    @abstractmethod
    def predict(self, X: np.ndarray) -> np.ndarray:
        pass
    
    @abstractmethod
    def compute_metrics(self, y_true: np.ndarray, y_pred: np.ndarray) -> Dict:
        pass
```

Now every subclass must implement all five methods. But they don't all need them.

- Regression: validates numerical arrays, preprocesses with normalization, trains, predicts continuous values, computes MAE/RMSE
- Classification: validates categorical labels, preprocesses with tokenization, trains, predicts class probabilities, computes precision/recall/F1
- Speech-to-text: validates audio buffers, preprocesses with resampling, trains, predicts text transcriptions, computes WER (word error rate)

They have nothing in common except the names. Speech-to-text shouldn't be forced to implement `compute_metrics()` the same way regression does. Classification's validation is nothing like regression's.

So you either:
1. Create empty stub methods (waste)
2. Throw `NotImplementedError` (runtime errors instead of design-time checks)
3. Create separate base classes for each pattern (defeats the purpose)

### Fragile Hierarchies

Do you inherit SentimentModel from ClassificationModel or BaseModel? SentimentModel *is* classification (binary/multi-class output), but has specialized postprocessing logic. If you inherit from ClassificationModel, you're committing to its contract. If SentimentModel later needs different preprocessing, you change ClassificationModel and potentially break other classifiers.

Or you duplicate the logic. Now you have two places to maintain the same code.

### Code Duplication

In practice, all validators do similar things (type checking, shape validation, range checking). All preprocessors normalize or tokenize. But because they're different types, you can't share them. You write MinMaxNormalizer for regression, then write a nearly identical MinMaxNormalizer for classification.

### Testing Nightmare

To test a single validator, you instantiate the full model object, mock the model function, mock the training data. The test bloats. You can't unit test the validator in isolation.

## The Shift: Composition

Instead of inheritance chains, compose the library from independent, focused components.

```python
# Define contracts as simple interfaces
class InputValidator:
    """Validates input data. Returns (is_valid, error_message)."""
    def validate(self, data: Dict) -> tuple[bool, str]:
        raise NotImplementedError

class Preprocessor:
    """Transforms raw data to model-ready features."""
    def process(self, data: Dict) -> Any:
        raise NotImplementedError

class Postprocessor:
    """Transforms model output to user-facing results."""
    def process(self, prediction: Any) -> Dict:
        raise NotImplementedError

class MetricsComputer:
    """Computes task-specific evaluation metrics."""
    def compute(self, y_true: np.ndarray, y_pred: np.ndarray) -> Dict:
        raise NotImplementedError
```

Now implement focused, reusable components:

```python
# Validators
class NumericalValidator:
    def __init__(self, shape: tuple, dtype: str = 'float32'):
        self.shape = shape
        self.dtype = dtype
    
    def validate(self, data: Dict) -> tuple[bool, str]:
        try:
            arr = np.array(data.get('features'), dtype=self.dtype)
            if arr.shape != self.shape:
                return False, f"Expected shape {self.shape}, got {arr.shape}"
            return True, ""
        except Exception as e:
            return False, str(e)

class TextValidator:
    def __init__(self, max_length: int = 512):
        self.max_length = max_length
    
    def validate(self, data: Dict) -> tuple[bool, str]:
        text = data.get('text', '')
        if not isinstance(text, str) or len(text) == 0:
            return False, "Text required"
        if len(text) > self.max_length:
            return False, f"Text too long (max {self.max_length})"
        return True, ""

class AudioValidator:
    def __init__(self, sample_rate: int = 16000):
        self.sample_rate = sample_rate
    
    def validate(self, data: Dict) -> tuple[bool, str]:
        audio = np.array(data.get('audio'))
        if audio.ndim != 1:
            return False, "Audio must be 1D"
        return True, ""

# Preprocessors
class MinMaxNormalizer:
    def __init__(self, min_val: float, max_val: float):
        self.min_val = min_val
        self.max_val = max_val
    
    def process(self, data: Dict) -> np.ndarray:
        features = np.array(data['features'], dtype='float32')
        return (features - self.min_val) / (self.max_val - self.min_val)

class TextTokenizer:
    def __init__(self, vocab_size: int = 10000):
        self.vocab_size = vocab_size
    
    def process(self, data: Dict) -> np.ndarray:
        tokens = [hash(word) % self.vocab_size for word in data['text'].split()]
        return np.array(tokens, dtype='int32')

class AudioResampler:
    def __init__(self, target_rate: int = 16000):
        self.target_rate = target_rate
    
    def process(self, data: Dict) -> np.ndarray:
        audio = np.array(data['audio'])
        # librosa.resample or similar
        return audio

# Postprocessors
class RegressionPostprocessor:
    def process(self, prediction: np.ndarray) -> Dict:
        return {"value": float(prediction[0])}

class ClassificationPostprocessor:
    def __init__(self, top_k: int = 3, threshold: float = 0.5):
        self.top_k = top_k
        self.threshold = threshold
    
    def process(self, prediction: np.ndarray) -> Dict:
        confidences = prediction[0]
        top_idx = int(np.argmax(confidences))
        top_conf = float(np.max(confidences))
        return {
            "predicted_class": top_idx,
            "confidence": top_conf,
            "pass_threshold": top_conf >= self.threshold
        }

class SentimentPostprocessor:
    def __init__(self, label_map: Dict[int, str] = None):
        self.label_map = label_map or {0: "negative", 1: "neutral", 2: "positive"}
    
    def process(self, prediction: np.ndarray) -> Dict:
        class_idx = int(np.argmax(prediction[0]))
        conf = float(np.max(prediction[0]))
        return {
            "sentiment": self.label_map[class_idx],
            "confidence": conf
        }

# Metrics Computers
class RegressionMetrics:
    def compute(self, y_true: np.ndarray, y_pred: np.ndarray) -> Dict:
        mae = np.mean(np.abs(y_true - y_pred))
        rmse = np.sqrt(np.mean((y_true - y_pred) ** 2))
        r2 = 1 - (np.sum((y_true - y_pred) ** 2) / np.sum((y_true - np.mean(y_true)) ** 2))
        return {"mae": mae, "rmse": rmse, "r2": r2}

class ClassificationMetrics:
    def compute(self, y_true: np.ndarray, y_pred: np.ndarray) -> Dict:
        accuracy = np.mean(y_true == y_pred)
        precision = self._precision(y_true, y_pred)
        recall = self._recall(y_true, y_pred)
        f1 = 2 * (precision * recall) / (precision + recall + 1e-9)
        return {"accuracy": accuracy, "precision": precision, "recall": recall, "f1": f1}
    
    def _precision(self, y_true, y_pred):
        tp = np.sum((y_true == 1) & (y_pred == 1))
        fp = np.sum((y_true == 0) & (y_pred == 1))
        return tp / (tp + fp + 1e-9)
    
    def _recall(self, y_true, y_pred):
        tp = np.sum((y_true == 1) & (y_pred == 1))
        fn = np.sum((y_true == 1) & (y_pred == 0))
        return tp / (tp + fn + 1e-9)

class SpeechToTextMetrics:
    def compute(self, y_true: list[str], y_pred: list[str]) -> Dict:
        # Word Error Rate (WER)
        wer_scores = [self._compute_wer(true, pred) for true, pred in zip(y_true, y_pred)]
        return {"wer": np.mean(wer_scores)}
    
    def _compute_wer(self, reference: str, hypothesis: str) -> float:
        # Simplified WER calculation
        ref_words = reference.split()
        hyp_words = hypothesis.split()
        errors = sum(1 for r, h in zip(ref_words, hyp_words) if r != h)
        errors += abs(len(ref_words) - len(hyp_words))
        return errors / len(ref_words) if ref_words else 0.0
```

Now compose endpoints by plugging components together:

```python
class MLModel:
    """A unified interface built from composed components."""
    
    def __init__(
        self,
        model_id: str,
        model_fn,  # Callable that does actual inference
        validator: InputValidator,
        preprocessor: Preprocessor,
        postprocessor: Postprocessor,
        metrics: MetricsComputer,
    ):
        self.model_id = model_id
        self.model_fn = model_fn
        self.validator = validator
        self.preprocessor = preprocessor
        self.postprocessor = postprocessor
        self.metrics = metrics
    
    def predict(self, data: Dict) -> Dict:
        valid, error = self.validator.validate(data)
        if not valid:
            raise ValueError(f"Invalid input: {error}")
        
        features = self.preprocessor.process(data)
        prediction = self.model_fn(features)
        result = self.postprocessor.process(prediction)
        result['model_id'] = self.model_id
        return result
    
    def evaluate(self, y_true: np.ndarray, y_pred: np.ndarray) -> Dict:
        return self.metrics.compute(y_true, y_pred)
```

Compose different model types with one interface:

```python
# Regression model
regression_model = MLModel(
    model_id='price_predictor_v2',
    model_fn=load_model('regression_v2.pkl'),
    validator=NumericalValidator(shape=(5,)),
    preprocessor=MinMaxNormalizer(min_val=0.0, max_val=100.0),
    postprocessor=RegressionPostprocessor(),
    metrics=RegressionMetrics()
)

# Classification model
classifier_model = MLModel(
    model_id='intent_classifier_v1',
    model_fn=load_model('classifier_v1.pkl'),
    validator=TextValidator(max_length=512),
    preprocessor=TextTokenizer(vocab_size=10000),
    postprocessor=ClassificationPostprocessor(top_k=3, threshold=0.6),
    metrics=ClassificationMetrics()
)

# Sentiment model
sentiment_model = MLModel(
    model_id='sentiment_v3',
    model_fn=load_model('sentiment_v3.pkl'),
    validator=TextValidator(max_length=256),
    preprocessor=TextTokenizer(vocab_size=5000),
    postprocessor=SentimentPostprocessor(),
    metrics=ClassificationMetrics()
)

# Speech-to-text model
stt_model = MLModel(
    model_id='speech_to_text_v1',
    model_fn=load_model('speech_to_text_v1.pkl'),
    validator=AudioValidator(sample_rate=16000),
    preprocessor=AudioResampler(target_rate=16000),
    postprocessor=RegressionPostprocessor(),
    metrics=SpeechToTextMetrics()
)

# All models use the same API
regression_model.predict({"features": [1.0, 2.0, 3.0, 4.0, 5.0]})
classifier_model.predict({"text": "what is the weather"})
sentiment_model.predict({"text": "this is great"})
stt_model.predict({"audio": audio_array})

# All models compute metrics the same way
regression_model.evaluate(y_true, y_pred_regression)
classifier_model.evaluate(y_true, y_pred_classification)
stt_model.evaluate(y_true_transcripts, y_pred_transcripts)
```

## Why This Works

**Extensibility.** Add a new model type (time-series, object detection, etc.) without changing existing code. Just compose a new set of components.

**Testability.** Test each component in isolation.

```python
def test_text_validator():
    v = TextValidator(max_length=256)
    valid, _ = v.validate({"text": "hello"})
    assert valid
    
    valid, msg = v.validate({"text": ""})
    assert not valid

def test_numerical_validator():
    v = NumericalValidator(shape=(5,))
    valid, _ = v.validate({"features": [1, 2, 3, 4, 5]})
    assert valid
    
    valid, msg = v.validate({"features": [1, 2, 3]})
    assert not valid

def test_text_tokenizer():
    t = TextTokenizer(vocab_size=1000)
    result = t.process({"text": "hello world"})
    assert result.shape == (2,)
```

No mocking the entire model stack. Tests run fast.

**Reusability.** Use the same component across different models:

```python
tokenizer = TextTokenizer(vocab_size=10000)

classifier_model = MLModel(
    ...,
    preprocessor=tokenizer,
    ...
)

sentiment_model = MLModel(
    ...,
    preprocessor=tokenizer,  # Reused
    ...
)
```

**Flexibility.** Combine components in ways a hierarchy can't express:

```python
class ChainedPreprocessor:
    def __init__(self, preprocessors: list[Preprocessor]):
        self.preprocessors = preprocessors
    
    def process(self, data: Dict) -> Any:
        result = data
        for preprocessor in self.preprocessors:
            result = preprocessor.process({"features": result})
        return result

# Chain: tokenize → normalize → embed
pipeline = ChainedPreprocessor([
    TextTokenizer(vocab_size=10000),
    MinMaxNormalizer(0, 10000),
])

advanced_model = MLModel(
    ...,
    preprocessor=pipeline,
    ...
)
```

## Integration with Training Code

This pattern extends to the full training lifecycle:

```python
class ModelTrainer:
    def __init__(self, model: MLModel):
        self.model = model
    
    def train(self, X_train: np.ndarray, y_train: np.ndarray, X_val: np.ndarray, y_val: np.ndarray):
        # Standard training loop
        for epoch in range(num_epochs):
            predictions = self.model.model_fn(X_train)
            loss = compute_loss(predictions, y_train)
            # Backprop, etc.
        
        # Evaluate on validation set
        val_predictions = self.model.model_fn(X_val)
        metrics = self.model.evaluate(y_val, val_predictions)
        return metrics

# Same trainer works for all model types
trainer = ModelTrainer(regression_model)
trainer.train(X_train, y_train, X_val, y_val)

trainer = ModelTrainer(classifier_model)
trainer.train(X_train, y_train, X_val, y_val)
```

## Lessons

**1. Avoid abstract base classes for heterogeneous systems.** If subclasses need to override nearly every method, you don't have a true hierarchy. You have a loose collection of things that happen to share a name.

**2. Use protocols or simple base classes.** Define what each component does, not a full contract everyone must implement.

**3. Test components in isolation.** Composition makes this practical. Use it.

**4. Expect composition overhead.** A few extra function calls per prediction. Negligible (<1ms) compared to model inference time. Profile before optimizing.

**5. Document component contracts clearly.** With inheritance, the contract is implicit in the class hierarchy. With composition, write docstrings.

## Production Impact

At Sway AI, this pattern reduced:
- Integration test failures: ~70% reduction (tests focus on individual components)
- Onboarding time for new model types: from 1 week to 1-2 days
- Code duplication: preprocessors and postprocessors reused across models
- Debugging time: single-responsibility components are easier to trace

The unified `predict()` and `evaluate()` API meant data scientists could swap models without changing application code. Engineers could add new metrics or validators without touching model-specific logic.

**The lesson: when you have diverse model types, composition scales. Inheritance doesn't.**