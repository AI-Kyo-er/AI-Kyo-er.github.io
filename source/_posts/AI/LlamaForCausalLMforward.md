---
title: LlamaForCausalLM.forward 被多繼承包裝，要重載庫函數來計算 流式指標?!
categories: [CS, AI]
tags:
- AI
date: 2025-08-22
---

## Baseline 写的很巧妙，但是真的好用吗？ Fuck 多继承！

### 多继承思路——Mixin
Mixin是一种程序设计中的多继承思路，一句话来说就是「我们想要给原有的类 A 添加新的 method， 所以先定义一个专门实现了这些新 method 的 Meta-method 类B（这个类通常不应该被直接实例化 ），然后将原有的类 和 Meta-method class B 进行多继承，形成一个新的类」。

EG. 1:
```python
class Vehicle(object):
    pass

class PlaneMixin(object):
    def fly(self):
        print('I am flying')

class Airplane(Vehicle, PlaneMixin):
    pass
```

为了方便维护 Airplane ，我们将它的独有的子类方法（飞行）放到一个新的方法类中，然后多继承让他拥有吧。

EG. 2:
<center><img src="./1.png"  width="55%"/></center>

一个抽象的例子，UML大家还能看懂吧。

### 什么？被这样包装后，函数接口和 Transformer.trainer 中使用的不一致了？

```python
def _get_output(...) -> Union[Tuple, CausalLMOutputWithPast]:
        if inputs_embeds is None:
            (
                input_ids,
                position_ids,
                attention_mask,
                past_key_values,
                inputs_embeds,
                labels,
            ) = self.prepare_inputs_labels_for_multimodal(
                input_ids, position_ids, attention_mask, past_key_values, labels, images, videos, audios
            )
        return super().forward(
            input_ids=input_ids,
            attention_mask=attention_mask,
            position_ids=position_ids,
            past_key_values=past_key_values,
            inputs_embeds=inputs_embeds,
            labels=labels,
            use_cache=use_cache,
            output_attentions=output_attentions,
            output_hidden_states=output_hidden_states,
            return_dict=return_dict
        )
    
    def forward(
        self,
        input_ids: torch.Tensor = None,
        attention_mask: Optional[torch.Tensor] = None,
        position_ids: Optional[torch.Tensor] = None,
        past_key_values: Optional[List[torch.Tensor]] = None,
        inputs_embeds: Optional[torch.Tensor] = None,
        labels: Optional[torch.Tensor] = None,
        use_cache: Optional[bool] = None,
        output_attentions: Optional[bool] = True,
        output_hidden_states: Optional[bool] = True,
        cache_position: Optional[torch.Tensor] = None,
        videos: Union[Optional[torch.Tensor], Optional[List[torch.Tensor]]] = None,
        return_dict: Optional[bool] = None,
    ) -> Union[Tuple, CausalLMOutputWithPast]:
        
        outputs = self._get_output(
            input_ids=input_ids,
            attention_mask=attention_mask,
            position_ids=position_ids,
            past_key_values=past_key_values,
            inputs_embeds=inputs_embeds,
            labels=labels,
            use_cache=use_cache,
            output_attentions=False,
            output_hidden_states=True,
            cache_position=cache_position,
            videos=videos,
            return_dict=return_dict,
        )
        loss = outputs.loss
        
        if not return_dict:
            output = (outputs.logits,) + outputs[1:]
            return (loss,) + output if loss is not None else output

        return CausalLMOutputWithPast(
            loss=loss,
            logits=outputs.logits,
            past_key_values=outputs.past_key_values,
            hidden_states=outputs.hidden_states,
            attentions=outputs.attentions,
        )
```

可以看到，作者将多模态信息的编码和对齐过程放在了 forward 函数中，封装简洁的同时也对程序员屏蔽了细节。

### 可惜，顺便对 Transformer.trainer 也屏蔽了细节......
Transformer.trainer 在 `__init__` 阶段可选地接受一个 `compute_metrics()`，来在 `Transformer.trainer.eval_loop` 中支持对生成式指标的计算。而生成式指标计算中，如果你没有给我你真正处理完的 labels，我拿着原始的纯文本的 labels ，**岂不是拔剑四顾心茫然？**

## 重载库函数
### 只能重载 `Transformer.trainer.eval_loop` 函数，让他将在 `forward()` 中处理过的真正的 labels 给到 `self.compute_metrics()`
```python
def prediction_step(
        self,
        model: nn.Module,
        inputs: Dict[str, Union[torch.Tensor, Any]],
        prediction_loss_only: bool,
        ignore_keys: Optional[List[str]] = None,
    ) -> Tuple[Optional[torch.Tensor], Optional[torch.Tensor], Optional[torch.Tensor]]:
    # ...

    final_labels = processed_labels if processed_labels is not None else original_labels
    if final_labels is not None:
        final_labels = nested_detach(final_labels)

    return (loss, logits, final_labels)


 def evaluation_loop(
        self,
        dataloader: DataLoader,
        description: str,
        prediction_loss_only: Optional[bool] = None,
        ignore_keys: Optional[List[str]] = None,
        metric_key_prefix: str = "eval",
    ) -> EvalLoopOutput:
    # ...

    all_preds = EvalLoopContainer(self.args.eval_do_concat_batches, padding_index=-100)
    losses, logits, labels = self.prediction_step(model, inputs, prediction_loss_only, ignore_keys=ignore_keys)
    all_labels.add(labels)
    all_preds.to_cpu_and_numpy()

    if args.include_inputs_for_metrics:
        metrics = self.compute_metrics(
            EvalPrediction(predictions=logits, label_ids=labels, inputs=inputs),
            compute_result=is_last_step,
        )
```

先继承 Trainer，然后将原实现复制进来，进行大刀阔斧的修改，顺便去掉我们用不到的特殊判断。

```python
from transformers import Trainer
from transformers.trainer import (
    is_sagemaker_mp_enabled,
    get_parameter_names,
    has_length,
    ALL_LAYERNORM_LAYERS,
    logger,
)
import datasets
import numpy as np
from typing import List, Optional

if is_torch_xla_available():
    import torch_xla.core.xla_model as xm

from motionepic.dataset.sampler import DistributedMultiDatasetBatchSampler
```

**但是 python 竟然命名空间上不像 cpp 一样会在继承的时候顺便继承父类的命名空间，所以要手动将这个子类中重载方法中用到的所有类内方法 import 进来。**

## 什么？直接OOM，实验室的服务器把我踢出登录，让我冷静半个小时不给登入？

原来有些生成式指标是不支持流式计算的，即需要将 `eval_dataset` 全部推理完，再将输出的 `pred`, `labels` 等丢进 `compute_metric()`，才能计算出准确的值。如 F1_score 就不像 acc 可以将每个独立的算出来然后求平均那么简单（是不等价的）。

所以 `Transformer.trainer.eval_loop` 的默认实现是将所有算完的先 `detach().cpu()` 放在内存中等着。。。天大的内存也不够你这么放啊，因为我是 MLLM 的输入输出，sequence 巨长。

### 添加流式计算
所以我们应该打开流式开关，并修改 `compute_metric()` 方法，让他在平时先计算每一个 eval_batch 的结果，仅仅将结果存着，而不是所有的 preds, labels. 在最后一个 eval_batch 算完后，再进行平均。

```python
def build_compute_metrics(tokenizer: transformers.PreTrainedTokenizer):
    # 类似于 cpp static 变量的东西
    accumulated_stats = {
        'bit_correct': 0,
        'bit_total': 0,
        'exact_match': 0,
        'samples': 0
    }
    
    def _compute_metrics(eval_preds, compute_result=False):
        predictions = eval_preds.predictions
        labels = eval_preds.label_ids

        logits = predictions[0] 

        labels_np = labels.cpu().numpy()
        logits_np = logits.cpu().numpy()
        
        pred_ids = np.argmax(logits_np, axis=-1) # (B, Seq, Vocab) -> (B, Seq)
        pred_ids = pred_ids[:, :-1]
        labels = labels_np[:, 1:]

        for i in range(labels.shape[0]):
            label_ids_i = labels[i][label_mask].tolist()
            pred_ids_i = pred_ids[i][pred_mask].tolist()

            label_text = tokenizer.decode(label_ids_i, skip_special_tokens=True)
            pred_text = tokenizer.decode(pred_ids_i, skip_special_tokens=True)

        # Accumulate batch statistics
        accumulated_stats['bit_correct'] += batch_bit_correct
        accumulated_stats['bit_total'] += batch_bit_total
        accumulated_stats['exact_match'] += batch_exact_match
        accumulated_stats['samples'] += batch_samples

        # Only return results when compute_result=True (last batch)
        if compute_result:
            # Reset accumulated stats for next evaluation
            final_stats = {
                "bit_match_accuracy": (accumulated_stats['bit_correct'] / accumulated_stats['bit_total']) if accumulated_stats['bit_total'] > 0 else 0.0,
                "exact_match_accuracy": (accumulated_stats['exact_match'] / accumulated_stats['samples']) if accumulated_stats['samples'] > 0 else 0.0,
                "num_samples": accumulated_stats['samples']
            }
            
            # Reset for next evaluation
            accumulated_stats['bit_correct'] = 0
            accumulated_stats['bit_total'] = 0
            accumulated_stats['exact_match'] = 0
            accumulated_stats['samples'] = 0
            
            return final_stats
        
        return None

    return _compute_metrics
```

其中，`compute_result` 就是`Transformer.trainer.eval_loop`给他的信号，如果是 `True` ，就是告诉他，这个是最后一个 batch 了，收拾收拾准备输出最后的结果吧。

**最终经过以上痛苦的重载过程，终于可以跑起来生成式指标的计算了，撒花。**