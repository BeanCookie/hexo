---
title: 基于HTTP和动态代理实现的简单RPC
date: 2020-10-21 09:27:50
categories:
- 设计模式
tags: 
- RPC
---

#### 现存问题

- 随着接口增多inner-api的rental模块中的接口变得愈发混乱，主要包括

  1：url定义不规范和由于多人开发存在功能有交叉的接口

  2：接口返回值没有文档规范或者其他能够明确描述的方式

  3：当同时调用多个inner-api接口时由于未做并发优化反映到调用端时会有响应过慢的问题

  4：开发时添加新接口会耗费比较多的精力重复编写模板代码，不够优雅

#### 优化思路

- 沿用HTTP协议借鉴RPC的设计思路通过代码封装简化现有的HTTP调用
- 优化调用的前提需要先规范接口url，在此基础之上才可以通过动态代理等方式根据类名以及方法名自动拼装出url简化HTTP调用
- 引用Gateway将PHP服务与内部服务隔离

#### 代码示例

```php
class ZHGTenantsClient
{

    /**
     * @param array $request
     * @param array $reply
     * @throws RPCClientNotImplementedException
     */
    function getAll(array $request, array $reply = ['contactPhone']) {
        throw new RPCClientNotImplementedException();
    }
}
```

```php
class ZHGAsyncClient
{

    /**
     * @param array $ids
     * @throws RPCClientNotImplementedException
     */
    public function getTenantAndProject(array $ids = ['tenantId' => null, 'projectId' => null]) {
        throw new RPCClientNotImplementedException();
    }

    /**
     * @param array $ids
     * @throws RPCClientNotImplementedException
     */
    public function getTenantAndProjectAndMachine(array $ids = ['tenantId' => null, 'projectId' => null, 'machineId' => 998]) {
        throw new RPCClientNotImplementedException();
    }
}
```

```php
class ReflectionTest extends TestCase
{
    /**
     * A basic test example.
     *
     * @return void
     */
    public function testGetMethods()
    {
        $proxy = RPCProxy::getInstance(ZHGTenantsClient::class);
        $reply = $proxy->getAll($request = ['contactPhone' => '13913801948']);
        Log::info('$reply', [$reply]);
    }
    
        /**
     * A basic test example.
     *
     * @return void
     */
    public function testZHGAsyncClient()
    {
        $proxy1 = RPCProxy::getInstance(ZHGAsyncClient::class);
        $reply1 = $proxy1->getTenantAndProject($ids = ['tenantId' => 1, 'projectId' => 1]);
        Log::info('$reply1', [$reply1]);
        Log::info('$reply1', [$reply1['tenant']]);
        Log::info('$reply1', [$reply1['tenant']->contactName]);
        Log::info('$reply1', [$reply1['project']]);
        Log::info('$reply1', [$reply1['project']->projectName]);

        $proxy2 = RPCProxy::getInstance(ZHGAsyncClient::class);
        $reply2 = $proxy2->getTenantAndProjectAndMachine($ids = ['tenantId' => 1, 'projectId' => 1, 'machineId' => 998]);
        Log::info('$reply2', [$reply2['machine']]);
        Log::info('$reply2', [$reply2['machine']->machineName]);
    }
}

```

```php
class RPCProxy
{
    // 真实代理对象
    private $target;

    //创建静态私有的变量保存该类对象

    static private $instance;
    //防止使用new直接创建对象

    private function __construct(){}
    //防止使用clone克隆对象

    private function __clone(){}

    static public function getInstance($targetClass)
    {
        //判断$instance是否是Singleton的对象，不是则创建
        if (!self::$instance instanceof self) {
            self::$instance = new self();
        }
        self::$instance->target = $targetClass;
        return self::$instance;
    }

    private function findRemoteMethodIndex($serviceAndMethodName)
    {
        return RPCUtils::getLastUpperCharIndex($serviceAndMethodName);
    }

    private function getRemoteHttpMethod($clientMethod)
    {
        $realMethodStartIndex = RPCUtils::getFirstUpperCharIndex($clientMethod);
        return strtoupper(substr($clientMethod, 0, $realMethodStartIndex));
    }

    private function getAsyncMethodNames($className)
    {
        $realMethodStartIndex = RPCUtils::getFirstUpperCharIndex($className);
        $methodNamesStr = substr($className, $realMethodStartIndex);
        return explode(RPCConst::ASYNC_METHOD_SPLIT, strtolower($methodNamesStr));
    }

    private function getRemoteOperation($clientMethod)
    {
        $realMethodStartIndex = RPCUtils::getLastUpperCharIndex($clientMethod);
        return strtoupper(substr($clientMethod, $realMethodStartIndex));
    }


    /**
     * @param $remoteHttpMethod
     * @param $serviceName
     * @param $remoteMethodName
     * @param $remoteOperation
     * @param $requestParam
     * @return \Psr\Http\Message\ResponseInterface
     * @throws GuzzleException
     */
    private function executeHttpRequest($remoteHttpMethod, $serviceName, $remoteMethodName, $remoteOperation, $requestParam)
    {

        $client = new Client();
        try {
            switch ($remoteOperation) {
                case OperationEnum::ALL:
                    return $client->request($remoteHttpMethod, RPCConst::GATEWAY_HOST . "/$serviceName/$remoteMethodName/", [
                        'json' => $requestParam,
                    ]);
                    break;
                case OperationEnum::ONE:
                    return $client->request($remoteHttpMethod, RPCConst::GATEWAY_HOST . "/$serviceName/$remoteMethodName/$requestParam");
                    break;
                default:
                    throw new CanNotFindOperationException();
            }
        } catch (GuzzleException $e) {
            throw $e;
        }
    }

    private function remoteMethodName2urlPath($remoteMethodName)
    {
        if (RPCUtils::strEndsWith($remoteMethodName, 's')) {
            return $remoteMethodName;
        }
        return $remoteMethodName . 's';
    }

    private function executeAsyncHttpRequest($remoteHttpMethod, $serviceName, array $remoteMethodNames, $requestParam)
    {
        $client = new Client(['base_uri' => RPCConst::GATEWAY_HOST]);

        $promises = collect();
        foreach ($remoteMethodNames as $remoteMethodName) {
            $remoteUrl = $this->remoteMethodName2urlPath($remoteMethodName);
            $requestId = $requestParam[$remoteMethodName . 'Id'];
            $promises->put($remoteMethodName, $client->getAsync("/$serviceName/$remoteUrl/$requestId"));
        }
        return unwrap($promises->toArray());
    }

    private function buildAsyncResponse($asyncHttpResp, $asyncMethodNames)
    {
        $jsonResp = collect();
        foreach ($asyncMethodNames as $asyncMethodName) {
            $jsonResp->put($asyncMethodName, (object)json_decode($asyncHttpResp[$asyncMethodName]->getBody(), true));
        }
        return $jsonResp->toArray();
    }

    private function buildResponse($httpResp, $remoteOperation, array $reply = null)
    {

        $jsonResp = json_decode($httpResp->getBody(), true);

        if ($remoteOperation === OperationEnum::ONE) {
            $jsonResp = [$jsonResp];
        }
        if ($reply) {
            $formatResp = collect($jsonResp)->map(function ($respItem) use ($reply) {
                return (object)array_filter($respItem, function ($val, $key) use ($reply) {
                    return in_array($key, $reply);
                }, ARRAY_FILTER_USE_BOTH);
            });
            if ($remoteOperation === OperationEnum::ONE) {
                return $formatResp->first();
            }
            return $formatResp->toArray();
        }
        return $jsonResp;
    }

    private function isAsyncMethod($className)
    {
        return strpos(strtolower($className), RPCConst::ASYNC_SIGN) !== false;
    }


    /**
     * @param $name
     * @param $args
     * @return array
     * @throws GuzzleException
     * @throws RPCClientParameterIllegalException
     * @throws \ReflectionException
     */
    public function __call($name, $args)
    {
        try {
            $ref = new ReflectionClass($this->target);

            // 获取类名
            $className = $ref->getShortName();

            $serviceAndMethodName = substr($className, 0, strpos($className, RPCConst::CLIENT_NAME));
            $lastMatch = $this->findRemoteMethodIndex($serviceAndMethodName);
            $serviceName = strtolower(substr($serviceAndMethodName, 0, $lastMatch));

            if ($this->isAsyncMethod($className)) {
                $asyncMethodNames = $this->getAsyncMethodNames($name);
                $remoteHttpMethod = strtoupper($this->getRemoteHttpMethod($name));
                $asyncHttpResp = $this->executeAsyncHttpRequest($remoteHttpMethod, $serviceName, $asyncMethodNames, $args[0]);
                return $this->buildAsyncResponse($asyncHttpResp, $asyncMethodNames);
            } else {
                $remoteMethodName = strtolower(substr($serviceAndMethodName, $lastMatch));
                $remoteHttpMethod = strtoupper($this->getRemoteHttpMethod($name));
                $remoteOperation = strtolower($this->getRemoteOperation($name));
                if (count($args) === 1) {
                    $realMethod = $ref->getMethod($name);
                    $parameters = $realMethod->getParameters();
                    if (count($parameters) === 2) {
                        $args[1] = $parameters[1]->getDefaultValue();
                    }
                }

                $httpResp = $this->executeHttpRequest($remoteHttpMethod, $serviceName, $remoteMethodName, $remoteOperation, $args[0]);
                return $this->buildResponse($httpResp, $remoteOperation, count($args) >= 2 ? $args[1] : null);
            }



        } catch (\ReflectionException $e) {
            throw $e;
        }
    }

}
```

