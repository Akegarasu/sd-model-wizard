<template>
  <div class="text-center p-6 max-w-1024px mx-auto">
    <h1 class="font-bold text-3xl">Stable Diffusion 模型解析</h1>
    <p class="text-gray-500 my-2 text-sm">自动判断模型类别</p>

    <div class="max-w-740px" style="margin: 0 auto">
      <el-upload class="upload-demo" drag multiple :before-upload="handleUpload">
        <el-icon class="el-icon--upload"><upload-filled /></el-icon>
        <div class="el-upload__text">拖动文件到这里或者点击上传</div>
      </el-upload>
    </div>

    <div v-if="fileRef" class="my-6">
      <div v-if="fileInfoRef" class="mt-4 text-left max-w-740px mx-auto">
        <h1 class="font-bold text-2xl mb-4">模型信息</h1>
        <div :class="[index === 0 && 'border-t border-t-gray-300']"
          class="bg-white border-b border-l border-r px-4 border-b-gray-300 border-l-gray-300 border-r-gray-300 py-2"
          v-for="(item, index) in fileInfoRef" :key="item.k">
          <h1 class="font-semibold text-sm text-gray-800">
            {{ item.k }}
          </h1>
          <p class="text-wrap break-all text-sm mt-1 text-gray-600" style="white-space: pre-wrap"
            v-if="item.k != 'Info'">
            {{ item.v }}
          </p>
          <json-viewer :value="jsonData" v-if="item.k == 'Info'"></json-viewer>
        </div>
      </div>
    </div>
    <p class="text-gray-500 my-2 text-sm">
      *运算完全在你的电脑上运行不会上传到云端
    </p>
    <a class="text-gray-500" href="https://www.bilibili.com/read/cv21362202">图文详解！最全模型用法</a>
    <div class="my-4 pt-4">
      如果您觉得本项目对您有帮助 请在 →
      <a class="inline-block text-sm text-gray-500" href="https://github.com/Akegarasu/sd-model-wizard">GitHub</a>
      ←上点个star
      <br />
      <span class="inline-block mt-2 text-sm text-gray-500">
        Made with ❤️ by
        <a class="text-gray-500" href="https://github.com/Akegarasu">@Akegarasu</a>
        <a> | </a>
        <a class="text-gray-500" href="https://space.bilibili.com/12566101">秋葉aaaki</a>
        <a> | </a>
        <a class="text-gray-500" href="https://novelai.dev">NovelAI.Dev</a>
      </span>
    </div>
  </div>
</template>

<script setup>
import { ElMessage } from "element-plus";
import { ref, watch } from "vue";
import jsonViewer from "vue-json-viewer";
import { UploadFilled, CopyDocument } from "@element-plus/icons-vue";

const fileRef = ref(null);
const fileInfoRef = ref(null);
const jsonData = ref(null);

const modelSig = {
  string_to_param: "Embedding",
  "model.diffusion_model.": "Stable Diffusion",
  "cond_stage_model.transformer.": "Stable Diffusion",
  lora_te_text_model_encoder: "LoRA",
  "encoder.down.0.block": "VAE",
  "linear.0.weight": "Hypernetworks",
  "linear1.weight": "Hypernetworks",
};

const modelUseGuide = {
  "Stable Diffusion":
    "大模型。放入 models/Stable-diffusion 文件夹后，进入 webui 在左上角点击刷新后选择模型。",
  VAE: "放入 models/VAE 文件夹后，在 webui 中的设置页面 - Stable Diffusion - 模型的 VAE 选择并保存",
  LoRA: "放入 models/Lora 文件夹后，在 webui 中，“生成” 按钮的下方选择 🎴 按钮，找到 Lora 选项卡点击使用。",
  Hypernetworks:
    "放入 models/hypernetworks 文件夹后，在 webui 中，“生成” 按钮的下方选择 🎴 按钮，找到 hypernetworks 选项卡点击使用。",
  Embedding:
    "放入 embeddings 文件夹后，在 webui 中，“生成” 按钮的下方选择 🎴 按钮，找到 embeddings 选项卡点击使用。",
};

watch(fileRef, () => {
  if (!fileRef.value) return;
  run();
});

async function handleUpload(file) {
  console.log(file);
  fileRef.value = file;
  return false;
}

const run = () => {
  const f = fileRef.value;
  const rd = new FileReader();
  rd.readAsBinaryString(f.slice(0, 1024 * 5));
  rd.onload = function (readRes) {
    // console.log(readRes.target.result);
    guessModel(readRes.target.result);
  };
};

const guessModel = (content) => {
  let modelType = "";
  let fileSize = fileRef.value.size;
  let fileExt = fileRef.value.name.split(".").pop();
  if (fileSize < 1024 * 10) {
    fileInfoRef.value = [{ k: "错误", v: "文件可能不是模型" }];
    return;
  }

  if (fileSize < 1024 * 1024 && content.indexOf("string_to_param") != -1) {
    modelType = "Embedding";
  } else {
    for (let sig in modelSig) {
      if (content.indexOf(sig) != -1) {
        modelType = modelSig[sig];
        break;
      }
    }
  }

  let modelTypeOk =
    modelType == "" ? "未知模型种类或非模型" : modelType + " 模型";
  let ok = [
    { k: "文件名", v: fileRef.value.name },
    { k: "文件大小", v: prettyBytes(fileSize) },
    { k: "模型种类", v: modelTypeOk },
  ];

  if (modelType != "") {
    ok.push({ k: "模型用法", v: modelUseGuide[modelType] });
  }

  if (fileExt == "safetensors" && modelType == "LoRA") {
    let ret = tryExtractLoraMeta(content);
    if (ret) {
      ok.push({ k: "Info", v: jsonData });
    }
  }

  fileInfoRef.value = ok;
};

const tryExtractLoraMeta = (content) => {
  const reg = new RegExp(/{"__metadata__":(.*ss_vae_name":.+?)}/);
  let match = reg.exec(content);
  if (match) {
    jsonData.value = JSON.parse(match[1] + "}");
    return true;
  }
  return false;
};

const prettyBytes = (size) => {
  const printable = (d, z) => {
    return `${d.toFixed(2)} ${z}`;
  };

  let kb = size / 1024;
  if (kb < 1024) {
    return printable(kb, "KB");
  }
  let mb = kb / 1024;
  if (mb < 1024) {
    return printable(mb, "MB");
  }

  let gb = mb / 1024;
  return printable(gb, "GB");
};
</script>
