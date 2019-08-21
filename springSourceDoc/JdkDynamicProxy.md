
# 【Spring源码】JAVA JDK动态代理及其在Spring源码中的应用
## 一.JDK动态代理Demo
使用JDK动态代理分三步走：
 1. 创建业务接口,并实现该接口对外提供服务   
    
    ```JAVA
        /**
         * MathCalculatorService
         * 创建业务接口
         * @author Vic
         * @date 2019/8/20
         */
        public interface MathCalculatorService {
        
            /**
             * add val1 and val2
             * @param val1
             * @param val2
             * @return
             */
            int add(int val1,int val2);
        
        }

    ```
    
    ```JAVA
      /**
       * MathCalculatorServiceImpl
       * 创建业务接口实现类
       * @author Vic
       * @date 2019/8/20
       */
      public class MathCalculatorServiceImpl implements MathCalculatorService {
      
          /**
           * {@link com.vic.demo.service.MathCalculatorService#add(int, int)}
           *
           * @param val1
           * @param val2
           * @return
           */
          @Override
          public int add(int val1, int val2) {
              System.out.println("---------------add-----------------");
              return val1 + val2;
          }
      }
  
    
    ```
 2. 创建自定义的InvocationHandler对接口提供的方法进行增强
 
    ```JAVA
        /**
         * MathInvocationHandler
         *
         * @author Vic
         * @date 2019/8/20
         */
        public class MathInvocationHandler implements InvocationHandler {
            /**
             * 目标对象
             */
            private Object target;
        
            public MathInvocationHandler(Object target) {
                super();
                this.target = target;
            }
        
            /**
             *  执行目标方法
             * @param proxy
             * @param method
             * @param args
             * @return
             * @throws Throwable
             */
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println("------------before---------------");
                Object result = method.invoke(target, args);
                System.out.println("------------after----------------");
                return result;
            }
            /**
             * 获取代理对象
             * @return
             */
            public Object getProxy() {
                return Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), target.getClass().getInterfaces(), this);
            }
        }

    ```
 3. 调用被代理的类

    ```java
        /**
         * ProxyTest
         *
         * @author Vic
         * @date 2019/8/20
         */
        
        public class ProxyTest {
        
            @Test
            public void testProxy() {
                // 实例化目标
                MathCalculatorService mathCalculatorService = new MathCalculatorServiceImpl();
                // 设置代理
                MathInvocationHandler mathInvocationHandler = new MathInvocationHandler(mathCalculatorService);
                // 获取被代理的对象
                MathCalculatorService proxy = (MathCalculatorService) mathInvocationHandler.getProxy();
                // 调用增强后的方法
                proxy.add(1, 1);
            }
        }

    ```
输出
```
        ------------before---------------
        ---------------add-----------------
        ------------after----------------
```

## JDK动态代理在Spring源码中的应用
    
刚刚InvocationHandler 创建的时候有3个函数:
* 构造函数，将被代理的对象传入
* invoke函数,此函数实现了AOP增强的所有逻辑
* getProxy函数，此函数写法大致相同，但是必不可少

定位到Spring 创建代理的代码 `org.springframework.aop.framework.DefaultAopProxyFactory.java`

    ```JAVA
    	@Override
    	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    		//optimize 用来控制CGLIB创建的代理是否使用激进的优化策略
    		// isProxyTargetClass 这个属性为true时，目标本身被代理 而不是目标类的接口
    		// 是否存在代理接口
    		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
    			Class<?> targetClass = config.getTargetClass();
    			if (targetClass == null) {
    				throw new AopConfigException("TargetSource cannot determine target class: " +
    						"Either an interface or a target is required for proxy creation.");
    			}
    			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
    				return new JdkDynamicAopProxy(config);
    			}
    			return new ObjenesisCglibAopProxy(config);
    		}
    		else {
    			return new JdkDynamicAopProxy(config);
    		}
    	}

    ```
spring 根据策略 选择使用JDK动态代理或者使用cglib，当选择使用JDK动态代理时 实例化了`JdkDynamicAopProxy` 对象我们看看`JdkDynamicAopProxy` 对象的实现吧！
果然`JdkDynamicAopProxy`  也实现了`InvocationHandler` 接口。 

我们找到了`getProxy()`方法 其同样使用了`Proxy.newProxyInstance` 方法,和刚刚的demo类似

    ```JAVA
        	@Override
        	public Object getProxy(@Nullable ClassLoader classLoader) {
        		if (logger.isTraceEnabled()) {
        			logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
        		}
        		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
        		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
        		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
        	}


    ```
既然其实现了InvocationHandler 接口 那么一定有`invoke`方法，我们看下`invoke` 的源码吧

    ```JAVA
        	/**
        	 * Implementation of {@code InvocationHandler.invoke}.
        	 * <p>Callers will see exactly the exception thrown by the target,
        	 * unless a hook method throws an exception.
        	 */
        	@Override
        	@Nullable
        	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        		MethodInvocation invocation;
        		Object oldProxy = null;
        		boolean setProxyContext = false;
        
        		TargetSource targetSource = this.advised.targetSource;
        		Object target = null;
        
        		try {
        			// equals 方法的处理
        			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
        				// The target does not implement the equals(Object) method itself.
        				return equals(args[0]);
        			}
        			// hash方法的处理
        			else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
        				// The target does not implement the hashCode() method itself.
        				return hashCode();
        			}
        			else if (method.getDeclaringClass() == DecoratingProxy.class) {
        				// There is only getDecoratedClass() declared -> dispatch to proxy config.
        				return AopProxyUtils.ultimateTargetClass(this.advised);
        			}
        			/**
        			 * class类的isAssignableFrom(Class clazz)  方法:
        			 *
        			 * 如果调用这个方法的class或接口与参数clazz表示的类或接口相同，
        			 * 或者是参数clazz表示的类或接口的父类则返回true
        			 */
        			else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
        					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
        				// Service invocations on ProxyConfig with the proxy config...
        				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
        			}
        
        			Object retVal;
        			// 有时候目标对象内部的自我调用将无法实施切面中的增强则需要通过此属性暴露代替
        			if (this.advised.exposeProxy) {
        				// Make invocation available if necessary.
        				oldProxy = AopContext.setCurrentProxy(proxy);
        				setProxyContext = true;
        			}
        
        			// Get as late as possible to minimize the time we "own" the target,
        			// in case it comes from a pool.
        			target = targetSource.getTarget();
        			Class<?> targetClass = (target != null ? target.getClass() : null);
        
        			// Get the interception chain for this method.
        			// 获取当前方法的拦截器链
        			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
        
        			// Check whether we have any advice. If we don't, we can fallback on direct
        			// reflective invocation of the target, and avoid creating a MethodInvocation.
        			if (chain.isEmpty()) {
        				// We can skip creating a MethodInvocation: just invoke the target directly
        				// Note that the final invoker must be an InvokerInterceptor so we know it does
        				// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
        
        				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
        				// 如果没有发现任何拦截器那么直接调用切点方法
        				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
        			}
        			else {
        				// We need to create a method invocation...
        				//将拦截器封装在ReflectiveMethodInvocation
        				//以便于使用其proceed 逐一对拦截器的调用
        				invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
        				// Proceed to the joinpoint through the interceptor chain.
        				//执行拦截器链
        				retVal = invocation.proceed();
        			}
        
        			// Massage return value if necessary.
        			Class<?> returnType = method.getReturnType();
        			//返回结果
        			if (retVal != null && retVal == target &&
        					returnType != Object.class && returnType.isInstance(proxy) &&
        					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
        				// Special case: it returned "this" and the return type of the method
        				// is type-compatible. Note that we can't help if the target sets
        				// a reference to itself in another returned object.
        				retVal = proxy;
        			}
        			else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
        				throw new AopInvocationException(
        						"Null return value from advice does not match primitive return type for: " + method);
        			}
        			return retVal;
        		}
        		finally {
        			if (target != null && !targetSource.isStatic()) {
        				// Must have come from TargetSource.
        				targetSource.releaseTarget(target);
        			}
        			if (setProxyContext) {
        				// Restore old proxy.
        				AopContext.setCurrentProxy(oldProxy);
        			}
        		}
        	}
      
    ```    
该函数的主要作用是创建了一个拦截器链，并使用`ReflectiveMethodInvocation`进行封装, 而`ReflectiveMethodInvocation`的`proceed`方法实现了拦截器的逐一调用

    ```JAVA
        	@Override
        	@Nullable
        	public Object proceed() throws Throwable {
        		//	We start with an index of -1 and increment early.
        		// 执行完所有增强后执行切点方法
        		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        			return invokeJoinpoint();
        		}
        		// 获取下一个要执行的拦截器
        		Object interceptorOrInterceptionAdvice =
        				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
        		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
        			// Evaluate dynamic method matcher here: static part will already have
        			// been evaluated and found to match.
        			//动态匹配
        			InterceptorAndDynamicMethodMatcher dm =
        					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
        			Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
        			if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
        				return dm.interceptor.invoke(this);
        			}
        			else {
        				// Dynamic matching failed.
        				// Skip this interceptor and invoke the next in the chain.
        				//不匹配则不执行拦截器
        				return proceed();
        			}
        		}
        		else {
        			// It's an interceptor, so we just invoke it: The pointcut will have
        			// been evaluated statically before this object was constructed.
        			// 普通拦截器直接执行
        			// e.g
        			// ExposeInvocationInterceptor
        			//DelegatePerTargetObjectIntroductionInterceptor
        			//MethodBeforeAdviceInterceptor
        			//AspectAroundAdvice
        			//AspectBeforeAdvice
        			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
        		}
        	}

        
    ```
`ReflectiveMethodInvocation` 维护着连接调用的计数器，记录了当前调用的位置，以便链接可以有序的进行下去,但是这个方法并没有维护各个增强器的顺序，而是交给了增强器本身进行逻辑实现
