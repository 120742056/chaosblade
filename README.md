# chaosblade代码结构分析（更详细部分见内网）
## 整体代码结构
1. ##### 上层根命令（blade）及子命令(prepare,revoke,create,destroy...)
1. ##### 注册二级子命令（cplus，jvm，cpu...）及三级子命令(cpu下fullload，k8s下container-cpu...)
1. ##### 在各场景执行器中，实现函数func (e *Executor) Exec(uid string, ctx context.Context, model *spec.ExpModel) *spec.Response，用来控制start或stop命令
1. ##### 底层具体实现逻辑
![整体结构](https://user-images.githubusercontent.com/3992234/56200113-bc1afe00-6070-11e9-82ef-860b68b14827.png)
## 各部分简单分析
### 上层根命令及子命令[Cli命令结构](https://github.com/chaosblade-io/chaosblade/blob/master/cli)
```
func CmdInit() *baseCommand {
	cli := NewCli()
	baseCmd := &baseCommand{
		command: cli.rootCmd,
	}

	....
	// add create command
	createCommand := &CreateCommand{}
	baseCmd.AddCommand(createCommand)

	// add destroy command
	destroyCommand := &DestroyCommand{}
	baseCmd.AddCommand(destroyCommand)
	...

	return baseCmd
}
```
### 注册构成二级子命令及三级子命令[代码地址](https://github.com/chaosblade-io/chaosblade/blob/master/cli/cmd/exp.go)
#### 注册所有场景
```
func (ec *baseExpCommandService) registerSubCommands() {
	// register os type command
	ec.registerOsExpCommands()
	// register jvm framework commands
	ec.registerJvmExpCommands()
	// register cplus
	ec.registerCplusExpCommands()
	// register docker command
	ec.registerDockerExpCommands()
	// register k8s command
	ec.registerK8sExpCommands()
}
```
#### 具体分析一个场景的注册（cplus为例）
##### 注册函数，其中主要逻辑，读取yaml文件，通过yaml和cplu执行器生成Model，通过Model注册二级命令和三级命令
```
// registerCplusExpCommands
func (ec *baseExpCommandService) registerCplusExpCommands() []*modelCommand {
	file := path.Join(util.GetBinPath(), "chaosblade-cplus-spec.yaml") //获取yaml文件
	models, err := util.ParseSpecsToModel(file, cplus.NewExecutor()) //根据yaml和cpuls.NewExecutor生成modle
	if err != nil {
		return nil
	}
	cplusCommands := make([]*modelCommand, 0) 
	for idx := range models.Models { //循环注册二级子命令（cpuls只有一个二级子命令cplus，os有多个二级子命令...）
		model := &models.Models[idx]
		command := ec.registerExpCommand(model, "") //循环二级子命了下所有三级子命令（delay，modify，return）
		cplusCommands = append(cplusCommands, command)
	}
	return cplusCommands
}

```
##### 根据yaml和executor生成Model（[ParseSpecsToModel](https://github.com/chaosblade-io/chaosblade-spec-go/blob/master/util/spec.go) ）
```
func ParseSpecsToModel(file string, executor spec.Executor) (*spec.Models, error) {
	if err != nil {
		return nil, err
	}
	models := &spec.Models{}
	err = yaml.Unmarshal(bytes, models)
	if err != nil {
		return nil, err
	}
	for idx := range models.Models {
		models.Models[idx].ExpExecutor = executor
	}
	return models, nil
}
```
###### [Model](https://github.com/chaosblade-io/chaosblade-spec-go/blob/master/spec/model.go)
```
type Models struct {
	Version string            `yaml:"version"`
	Kind    string            `yaml:"kind"`
	Models  []ExpCommandModel `yaml:"items"`
}
type ExpCommandModel struct {
	ExpName         string          `yaml:"target"`
	ExpShortDesc    string          `yaml:"shortDesc"`
	ExpLongDesc     string          `yaml:"longDesc"`
	ExpExample      string          `yaml:"example"`
	ExpActions      []ActionModel   `yaml:"actions"`
	ExpExecutor     Executor        `yaml:"-"`
	ExpFlags        []ExpFlag       `yaml:"flags,omitempty"`
	ExpScope        string          `yaml:"scope"`
	ExpPrepareModel ExpPrepareModel `yaml:"prepare,omitempty"`
	ExpSubTargets   []string        `yaml:"subTargets,flow,omitempty"`
}
```
###### yaml文件结构（[Cplus为例](https://github.com/chaosblade-io/chaosblade-exec-cplus/blob/master/src/main/resources/chaosblade-cplus-spec.yaml)）
```
kind: plugin
version: v1
items:
- target: cplus //二级子命令 blade create cplus ..........
  shortDesc: c++ experiment
  longDesc: c++ experiment for testing program delay, variable changed and method return.
  example: cplus delay --delayDuration 5 --breakLine tcp_server.cpp:39 --fileLocateAndName /home/admin/socketServer/main --forkMode child --processName main --delayDuration 5 --initParams 9527 --libLoad /home/lib
  actions:
  - action: delay //命令执行动作 blade create cplus delay .........
    aliases: []
    flags: //命令参数 
    - desc: delay time
      name: delayDuration
      noArgs: false
      required: true
    longDesc: delay time...
    matchers://命令匹配规则
    - desc: Injection line in source code
      name: breakLine
      noArgs: false
      required: true
    ....
    shortDesc: delay time

...
```
###### 执行器
所有执行器需要实现如下接口
```
func (e *Executor) Exec(uid string, ctx context.Context, model *spec.ExpModel) *spec.Response {
    ...
    // 在函数中利用spec.IsDestroy(ctx)判断启动实验或销毁实验，例如
    if _, ok := spec.IsDestroy(ctx); ok {
	//调用停止实验函数，向可执行文件或url发送参数和停止命令
    } else {
        //启动实验实验函数，向可执行文件或url发送参数和启动命令
    }
}
```
以cpuls为例（[executor.go](https://github.com/chaosblade-io/chaosblade/blob/master/exec/cplus/executor.go)）
```
func (e *Executor) Exec(uid string, ctx context.Context, model *spec.ExpModel) *spec.Response {
	...
	if _, ok := spec.IsDestroy(ctx); ok {
		url = e.destroyUrl(port, uid)
	} else {
		url = e.createUrl(port, uid, model)
	}
	result, err, code := util.Curl(url)
	....
}
```

### 场景底层具体实现逻辑（cplus和os，k8s为例）
#### cpuls场景主逻辑
1. ###### cplus执行器根据spec.IsDestroy(ctx)获取启动或停止url，通过Curl访问接口
1. ###### 利用java application实现对外接口，逻辑控制
1. ###### 通过调用工厂获取对应ModelService实例,调用实例函数执行并获取结果
1. ###### ModelService执行时分为追踪已有进程（有pid）或新启进程（无pid）调用底层脚本
1. ###### 底层通过expect和gdb自动交互,gdb通过设置断点，注入命令，实现故障
##### cplus执行器，实现Exec函数（所有执行需要实现该函数）（[executor.go](https://github.com/chaosblade-io/chaosblade/blob/master/exec/cplus/executor.go)）
```
func (e *Executor) Exec(uid string, ctx context.Context, model *spec.ExpModel) *spec.Response {
	var url string
	...
	if _, ok := spec.IsDestroy(ctx); ok {
		url = e.destroyUrl(port, uid)
	} else {
		url = e.createUrl(port, uid, model)
	}
	result, err, code := util.Curl(url)
	...
}
```
##### java application整体结构不多赘述，分析CreateController([CreateController.java](https://github.com/chaosblade-io/chaosblade-exec-cplus/blob/master/src/main/java/com/alibaba/chaosblade/exec/cplus/controller/CreateController.java))
```
@RestController
public class CreateController {

    ...
    @RequestMapping("/create")
        ...
        // 检查输入参数格式是否正确
        ...       
        //根据工厂生成ModelService
        final IModelService iModelService = modelFactory.createModelService(actionArg);
        ...
                                    @Override
                                    public void run() {
                                        //执行ModelService
                                        Result injectionResult = iModelService.handleInjection(requestParamsBean);
                                    }


```
##### ModelFactory，根据输入参数返回Action对应ModelService实例([ModelFactory.java](https://github.com/chaosblade-io/chaosblade-exec-cplus/blob/master/src/main/java/com/alibaba/chaosblade/exec/cplus/service/ModelFactory.java))
```
public class ModelFactory {

    @Autowired
    private DelayModelService delayModelService;
    ...

    public IModelService createModelService(String actionName) {
        if(Constants.DELAY_ACTION_NAME.equals(actionName)){
            return delayModelService;
        } else if{
        ...
    }
}
```
##### 执行，ModelService根据pid是否为空（追踪存在进程或新启进程），执行并返回脚本执行结果([DelayModelService.java](https://github.com/chaosblade-io/chaosblade-exec-cplus/blob/master/src/main/java/com/alibaba/chaosblade/exec/cplus/service/DelayModelService.java))
```
public Result handleInjection(RequestParamsBean requestParamsBean) {
        ...
        if (StringUtil.isBlank(pid)){
            return ExecShellUtils.execShell(strScriptLocation + strScriptDelayFileName, ...);
        } else {
            return ExecShellUtils.execShell(strScriptLocation + strScriptDelayAttachFileName, ...);
        }
    }
```
#####  通过expect和gdb交互，gdb通过设置断点，注入命令，执行故障([shell_response_delay.sh](https://github.com/chaosblade-io/chaosblade-exec-cplus/blob/master/src/main/resources/script/shell_response_delay.sh))
```
#!/bin/bash

expect -c "
  spawn gdb
  expect {
    \"gdb\" {send \"file $1\n\";} //通过文件调试进程
  }
  ...
    \"gdb\" {send \"b $4\n\";} //在$4的位置设置断点
  }
  expect {
    \"gdb\" {send \"commands\n\";} //设置断点执行的命令
  }
  expect {
    \">\" {send \"shell sleep $5\n\"} //延时$5秒
  }
  expect {
    \">\" {send \"cont\n\"} //继续执行
  }
  expect {
    \">\" {send \"end\n\"} //结束断点命令
  }
  expect {
    \"gdb\" {send \"r $7\n\";} //启动gdb调试
  }
 interact
'''
```
### java类似，通过jvm实现各注入故障
### os场景分析，主逻辑，
1. ###### chaosblade-exe-os代码库中注册各场景命令，生成yaml文件(用于chaosblade库中生成命令）
1. ###### 每个子命令有自己的执行器
1. ###### 执行器中通过spec.IsDestroy(ctx)执行启动或停止命令
1. ###### 启动或停止命令中通过cli获取的参数，传递参数给对应可执行文件
1. ###### 编译时通过*.go生成可执行文件而来
#### chaosblade-exe-os生成yaml文件([spec.go](https://github.com/chaosblade-io/chaosblade-exec-os/blob/master/build/spec.go))
```
func main() {
	...
	err := util.CreateYamlFile(getModels(), os.Args[1])
	...
}
func getModels() *spec.Models {
	modelCommandSpecs := []spec.ExpModelCommandSpec{
		exec.NewCpuCommandModelSpec(),
		exec.NewMemCommandModelSpec(),
		exec.NewProcessCommandModelSpec(),
		exec.NewNetworkCommandSpec(),
		exec.NewDiskCommandSpec(),
		exec.NewScriptCommandModelSpec(),
	}
	...
	return util.MergeModels(specModels...)
}
```
#### network场景,子命令drop为例
##### network场景注册子命令([network.go](https://github.com/chaosblade-io/chaosblade-exec-os/blob/master/exec/network.go))
```
func NewNetworkCommandSpec() spec.ExpModelCommandSpec {
	return &NetworkCommandSpec{
		spec.BaseExpModelCommandSpec{
			ExpActions: []spec.ExpActionCommandSpec{
				NewDelayActionSpec(),
				NewDropActionSpec(),
				NewDnsActionSpec(),
				NewLossActionSpec(),
				NewDuplicateActionSpec(),
				NewCorruptActionSpec(),
				NewReorderActionSpec(),
				NewOccupyActionSpec(),
			},
			ExpFlags: []spec.ExpFlagSpec{},
		},
	}
}
```
##### 子命令，实现Exec函数、对应的启动停止函数([network_drop.go](https://github.com/chaosblade-io/chaosblade-exec-os/blob/master/exec/network_drop.go))
###### Exec() 
```
func (ne *NetworkDropExecutor) Exec(suid string, ctx context.Context, model *spec.ExpModel) *spec.Response {
	...
	localPort := model.ActionFlags["local-port"]
	remotePort := model.ActionFlags["remote-port"]
	if _, ok := spec.IsDestroy(ctx); ok {
		return ne.stop(localPort, remotePort, ctx)
	}

	return ne.start(localPort, remotePort, ctx)
}
```
###### start()
```
func (ne *NetworkDropExecutor) start(localPort, remotePort string, ctx context.Context) *spec.Response {
	args := fmt.Sprintf("--start --debug=%t", util.Debug)
	if localPort != "" {
		args = fmt.Sprintf("%s --local-port %s", args, localPort)
	}
	if remotePort != "" {
		args = fmt.Sprintf("%s --remote-port %s", args, remotePort)
	}
	return ne.channel.Run(ctx, path.Join(ne.channel.GetScriptPath(), dropNetworkBin), args) //通过传参执行可执行文件实现故障
}
```
###### stop()
```
func (ne *NetworkDropExecutor) stop(localPort, remotePort string, ctx context.Context) *spec.Response {
	args := fmt.Sprintf("--stop --debug=%t", util.Debug)
	if localPort != "" {
		args = fmt.Sprintf("%s --local-port %s", args, localPort)
	}
	if remotePort != "" {
		args = fmt.Sprintf("%s --remote-port %s", args, remotePort)
	}
	return ne.channel.Run(ctx, path.Join(ne.channel.GetScriptPath(), dropNetworkBin), args) //通过传参执行可执行文件停止故障
}
```
##### dropNetworkBin，底部利用iptables添加规则实现故障，删除规则停止故障（[dropnetwork.go](https://github.com/chaosblade-io/chaosblade-exec-os/blob/master/exec/bin/dropnetwork/dropnetwork.go)）
```
func handleDropSpecifyPort(remotePort string, localPort string, ctx context.Context) {
	var response *spec.Response
	if localPort != "" {
		response = cl.Run(ctx, "iptables",
                        //-A INPUT:对input添加规则
                        //-p tcp 匹配tcp协议，
                        // --dport localPort 匹配localPort端口
                        // -j DROP 动作DROP（丢包）
			fmt.Sprintf(`-A INPUT -p tcp --dport %s -j DROP`, localPort))
		...
	}
	...
	bin.PrintOutputAndExit(response.Result.(string))
}
```
```
func stopDropNet(localPort, remotePort string) {
	ctx := context.Background()
	if localPort != "" {
                //-D INPUT:对input删除规则
                //-p tcp 匹配tcp协议，
                //--dport localPort 匹配localPort端口
                //-j DROP 动作DROP（丢包）
		cl.Run(ctx, "iptables", fmt.Sprintf(`-D INPUT -p tcp --dport %s -j DROP`, localPort))
		...
	}
}
```
### k8s场景分析
#### k8s主逻辑
1. ##### k8
