# Deferred-Rendering-SSAO-in-unity-URP

![Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled.png](Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled.png)

![Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%201.png](Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%201.png)

该项目为使用Deferred render path的URP加入SSAO的支持，修改成延迟渲染后，发现URP原来的SSAO贴图的渲染是正常的，但是最终画面没有SSAO效果：

![Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%202.png](Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%202.png)

所有应该是在延迟渲染的光照计算中没有考虑SSAO，考虑在Fragment Shader中直接采样贴图实现，先要找到URP的Deferred中计算光照的位置。

![Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%203.png](Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%203.png)

直接参考Forward中的SSAO实现方式修改代码：

```cpp
// "Hidden/Universal Render Pipeline/StencilDeferred"
half4 DeferredShading(Varyings input) : SV_Target
	{
		UNITY_SETUP_INSTANCE_ID(input);
		UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX(input);

		// Using SAMPLE_TEXTURE2D is faster than using LOAD_TEXTURE2D on iOS platforms (5% faster shader).
		// Possible reason: HLSLcc upcasts Load() operation to float, which doesn't happen for Sample()?
		float2 screen_uv = (input.screenUV.xy / input.screenUV.z);
		float d = SAMPLE_TEXTURE2D_X_LOD(_CameraDepthTexture, my_point_clamp_sampler, screen_uv, 0).x;
		// raw depth value has UNITY_REVERSED_Z applied on most platforms.
		half4 gbuffer0 = SAMPLE_TEXTURE2D_X_LOD(_GBuffer0, my_point_clamp_sampler, screen_uv, 0);
		half4 gbuffer1 = SAMPLE_TEXTURE2D_X_LOD(_GBuffer1, my_point_clamp_sampler, screen_uv, 0);
		half4 gbuffer2 = SAMPLE_TEXTURE2D_X_LOD(_GBuffer2, my_point_clamp_sampler, screen_uv, 0);
//SSAO here
		#if defined(_SCREEN_SPACE_OCCLUSION)
		AmbientOcclusionFactor aoFactor;
//采样SSAO贴图
		aoFactor.indirectAmbientOcclusion = SAMPLE_TEXTURE2D_X_LOD(_ScreenSpaceOcclusionTexture, my_point_clamp_sampler,screen_uv, 0);
		aoFactor.directAmbientOcclusion = lerp(1.0, aoFactor.indirectAmbientOcclusion, _AmbientOcclusionParam.w);
		#endif

		#ifdef _DEFERRED_SUBTRACTIVE_LIGHTING
        half4 gbuffer4 = SAMPLE_TEXTURE2D_X_LOD(_GBuffer4, my_point_clamp_sampler, screen_uv, 0);
        half4 shadowMask = gbuffer4;
		#else
		half4 shadowMask = 1.0;
		#endif
......
// 参考Forward中的方式，使得AO影响直接光和间接光
#if defined(_SCREEN_SPACE_OCCLUSION)
		// 直接光：
		unityLight.color *= aoFactor.directAmbientOcclusion;
		// 间接光：
		inputData.bakedGI *= aoFactor.indirectAmbientOcclusion;
#endif
```

但是修改后发现相关跟Forward相比更加明亮：

![Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%204.png](Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%204.png)

仔细研究源码发现，inputData.bakedGI *= aoFactor.indirectAmbientOcclusion; 这行代码是不起作用的，在StencilDeferred中inputData是一直为零的，那么GI去了哪里呢？

```cpp
FragmentOutput output;
    output.GBuffer0 = half4(brdfData.diffuse.rgb, PackMaterialFlags(materialFlags)); // diffuse         diffuse         diffuse         materialFlags   (sRGB rendertarget)
    output.GBuffer1 = half4(specular, brdfData.reflectivity);                        // specular        specular        specular        reflectivity    (sRGB rendertarget)
    output.GBuffer2 = half4(packedNormalWS, packedSmoothness);                       // encoded-normal  encoded-normal  encoded-normal  smoothness
    output.GBuffer3 = half4(globalIllumination, 0);                                  // GI              GI              GI              [not_available] (lighting buffer)
```

GI是保存在第四个GBuffer里面的，我们来找找urp对GBuffer3的操作：

![Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%205.png](Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%205.png)

？？？只有五行提到了？

再来看看StencilDeferred中的InputData是如何提取的：

```cpp
InputData InputDataFromGbufferAndWorldPosition(half4 gbuffer2, float3 wsPos)
{
    InputData inputData;

    inputData.positionWS = wsPos;

#if _GBUFFER_NORMALS_OCT
    half2 remappedOctNormalWS = Unpack888ToFloat2(gbuffer2.xyz); // values between [ 0,  1]
    half2 octNormalWS = remappedOctNormalWS.xy * 2.0h - 1.0h;    // values between [-1, +1]
    inputData.normalWS = UnpackNormalOctQuadEncode(octNormalWS);
#else
    inputData.normalWS = normalize(gbuffer2.xyz);  // values between [-1, +1]
#endif

    inputData.viewDirectionWS = SafeNormalize(GetWorldSpaceViewDir(wsPos.xyz));

    // TODO: pass this info?
    inputData.shadowCoord     = (float4)0;
    inputData.fogCoord        = (half  )0;
    inputData.vertexLighting  = (half3 )0;

    inputData.bakedGI = (half3)0; // Note: this is not made available at lighting pass in this renderer - bakedGI contribution is included (with emission) in the value GBuffer3.rgb, that is used as a renderTarget during lighting

    return inputData;
}
```

原来GBuffer3是作为光照计算的renderTarget ，怪不得在光照计算的shader中全程没有提到GBuffer3了！

GBuffer3作为RenderTarget：

```cpp
// ScriptableRenderPass
        public override void Configure(CommandBuffer cmd, RenderTextureDescriptor cameraTextureDescripor)
        {
// GBuffer3 -> GI -> m_DeferredLights.GbufferAttachmentIdentifiers[m_DeferredLights.GBufferLightingIndex]
            RenderTargetIdentifier lightingAttachmentId = m_DeferredLights.GbufferAttachmentIdentifiers[m_DeferredLights.GBufferLightingIndex];
            RenderTargetIdentifier depthAttachmentId = m_DeferredLights.DepthAttachmentIdentifier;

            // TODO: Cannot currently bind depth texture as read-only!
            ConfigureTarget(lightingAttachmentId, depthAttachmentId);
        }

        // ScriptableRenderPass
        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
//在G-Buffer 3 的基础上进行其他光照的计算
            m_DeferredLights.ExecuteDeferredPass(context, ref renderingData);
        }
```

看起来好像是先计算GI，然后再把其他光源和GI进行叠加，再来看看其他光源是怎么计算和叠加的：

```cpp
// DeferredLights.cs
void RenderStencilDirectionalLights(CommandBuffer cmd, ref RenderingData renderingData, NativeArray<VisibleLight> visibleLights, int mainLightIndex)
        {
            cmd.EnableShaderKeyword(ShaderKeywordStrings._DIRECTIONAL);

            // Directional lights.

            // TODO bundle extra directional lights rendering by batches of 8.
            // Also separate shadow caster lights from non-shadow caster.
            for (int soffset = m_stencilVisLightOffsets[(int)LightType.Directional]; soffset < m_stencilVisLights.Length; ++soffset)
            {
                ushort visLightIndex = m_stencilVisLights[soffset];
                VisibleLight vl = visibleLights[visLightIndex];
                if (vl.lightType != LightType.Directional)
                    break;

                Vector4 lightDir, lightColor, lightAttenuation, lightSpotDir, lightOcclusionChannel;
                UniversalRenderPipeline.InitializeLightConstants_Common(visibleLights, visLightIndex, out lightDir, out lightColor, out lightAttenuation, out lightSpotDir, out lightOcclusionChannel);

                int lightFlags = 0;
                if (vl.light.bakingOutput.lightmapBakeType == LightmapBakeType.Mixed)
                    lightFlags |= (int)LightFlag.SubtractiveMixedLighting;

                // Setup shadow paramters:
                // - for the main light, they have already been setup globally, so nothing to do.
                // - for other directional lights, it is actually not supported by URP, but the code would look like this.
                bool hasDeferredShadows;
                if (visLightIndex == mainLightIndex)
                {
                    hasDeferredShadows = vl.light && vl.light.shadows != LightShadows.None;
                    cmd.DisableShaderKeyword(ShaderKeywordStrings._DEFERRED_ADDITIONAL_LIGHT_SHADOWS);
                }
                else
                {
                    int shadowLightIndex = m_AdditionalLightsShadowCasterPass != null ? m_AdditionalLightsShadowCasterPass.GetShadowLightIndexFromLightIndex(visLightIndex) : -1;
                    hasDeferredShadows = vl.light && vl.light.shadows != LightShadows.None && shadowLightIndex >= 0;
                    CoreUtils.SetKeyword(cmd, ShaderKeywordStrings._DEFERRED_ADDITIONAL_LIGHT_SHADOWS, hasDeferredShadows);

                    cmd.SetGlobalInt(ShaderConstants._ShadowLightIndex, shadowLightIndex);
                }

                bool hasSoftShadow = hasDeferredShadows && renderingData.shadowData.supportsSoftShadows && vl.light.shadows == LightShadows.Soft;
                CoreUtils.SetKeyword(cmd, ShaderKeywordStrings.SoftShadows, hasSoftShadow);

                cmd.SetGlobalVector(ShaderConstants._LightColor, lightColor); // VisibleLight.finalColor already returns color in active color space
                cmd.SetGlobalVector(ShaderConstants._LightDirection, lightDir);
                cmd.SetGlobalInt(ShaderConstants._LightFlags, lightFlags);

                // Lighting pass.
                cmd.DrawMesh(m_FullscreenMesh, Matrix4x4.identity, m_StencilDeferredMaterial, 0, 3); // Lit
                cmd.DrawMesh(m_FullscreenMesh, Matrix4x4.identity, m_StencilDeferredMaterial, 0, 4); // SimpleLit
            }

            cmd.DisableShaderKeyword(ShaderKeywordStrings._DEFERRED_ADDITIONAL_LIGHT_SHADOWS);
            cmd.DisableShaderKeyword(ShaderKeywordStrings.SoftShadows);
            cmd.DisableShaderKeyword(ShaderKeywordStrings._DIRECTIONAL);
        }
```

奇怪的是，在得到的最总结果中，光照是绘制了一个全屏的面片：

```cpp
// Lighting pass.
                cmd.DrawMesh(m_FullscreenMesh, Matrix4x4.identity, m_StencilDeferredMaterial, 0, 3); // Lit
                cmd.DrawMesh(m_FullscreenMesh, Matrix4x4.identity, m_StencilDeferredMaterial, 0, 4); // SimpleLit
```

而他的输出是不透明的：

![Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%206.png](Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%206.png)

这不就直接覆盖了吗，怎么做到的混合呢？

![Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%207.png](Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%207.png)

原来是再shader里面设置了Blend命令进行了混合，终于可以梳理下整个流程了：

![Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%208.png](Deferred-Rendering-SSAO-in-unity-URP%2050a9fb598304471bb1f12e565a13b5ef/Untitled%208.png)