## host-local
代码架构与cni plugin 架构类似，也是使用 command 形式， 主要包括 cmdAdd 与 cmdDel.
github 地址：https://github.com/containernetworking/plugins

### 代码流程
    
#### 注册环节
   在main.go里， 将相应的的操作func注册
   
```
func main() {
    // TODO: implement plugin version
    skel.PluginMain(cmdAdd, cmdGet, cmdDel, version.All, "TODO")
}
```

#### cmdAdd
##### 配置初始化（LoadIPAMConfig） 

* 首先将ipam config 文件内容转换成 Net struct 
* 根据Net struct 将对应ip range 加入 ranges

ipamconfig config 结构体
```
type IPAMConfig struct {
    *Range
    Name       string
    Type       string         `json:"type"`
    Routes     []*types.Route `json:"routes"`
    DataDir    string         `json:"dataDir"`
    ResolvConf string         `json:"resolvConf"`
    Ranges     []RangeSet     `json:"ranges"`
    IPArgs     []net.IP       `json:"-"` // Requested IPs from CNI_ARGS and args
}

type RangeSet []Range
```
Range 结构体
```
type Range struct {
    RangeStart net.IP      `json:"rangeStart,omitempty"` // The first ip, inclusive
    RangeEnd   net.IP      `json:"rangeEnd,omitempty"`   // The last ip, inclusive
    Subnet     types.IPNet `json:"subnet"`
    Gateway    net.IP      `json:"gateway,omitempty"`
}
```
根据结构体可以分析出IPAMConfig.Range 与 IPAMConfig.Ranges对应的数组单元是一致的， 在loadIPAMConfig func 中会将IPAMConfig.Range append到IPAMConfig.Ranges[]中；
故通常我们看配置文件，通常是没有ranges参数的，怀疑是给特别的cni插件使用，配置多ip

##### 配置存储ip地址信息的宿主机目录地址（disk.New）
从中就说明一个问题， host-local不会根据当前宿主机的容器ip 自动筛选， 当删除容器异常或者docker异常事故， 功能导致存储在该目录下文件没有删除，从而导致ip耗尽
```
const lastIPFilePrefix = "last_reserved_ip."
const LineBreak = "\r\n"

var defaultDataDir = "/var/lib/cni/networks"

// Store implements the Store interface
var _ backend.Store = &Store{}

func New(network, dataDir string) (*Store, error) {
    if dataDir == "" {
        dataDir = defaultDataDir
    }
    dir := filepath.Join(dataDir, network)
    if err := os.MkdirAll(dir, 0755); err != nil {
        return nil, err
    }

    lk, err := NewFileLock(dir)
    if err != nil {
        return nil, err
    }
    return &Store{lk, dir}, nil
}
```
查看代码发现，其存储ip信息的具体目录为/var/lib/cni/networks，若ipam无name，则该目录为根目录，若ipam有name ，则目录层级为/var/lib/cni/network/{ipam.name}

##### 根据Ranges的配置来获取对应ip
以ranges slice 为循环操作， 发挥ips

* 获取index ， value(ranges[index])
* 去/var/lib/cni/network/{ipam.name} 目录获取last_reserved_ip.{index} 文件内容，确定最后一次ip
* 按照顺位，获取下一位ip；在/var/lib/cni/network/{ipam.name} 目录下创建生成 名为{ip}的文件，并将containerId写入该文件；更新last_reserved_ip.{index}，将{ip}更新进去
    * 在获取ip阶段，有一层for死循环，若出现创建{ip}文件已存在，则重新进行顺位，获取下一位，知道循环回上一次ip，则认为无ip资源可用

```
iter, err := a.GetIter()
if err != nil {
    return nil, err
}
for {
    reservedIP, gw = iter.Next()
    if reservedIP == nil {
        break
	}

    reserved, err := a.store.Reserve(id, ifname, reservedIP.IP, a.rangeID)
    if err != nil {
        return nil, err
    }

    if reserved {
        break
    }
}
```
当根据新的ip在目录下，发现已经有该文件创建的目录，则在a.store.Reserve， 返回 `false ， nil`
```
f, err := os.OpenFile(fname, os.O_RDWR|os.O_EXCL|os.O_CREATE, 0644)
if os.IsExist(err) {
    return false, nil
}
```
这样就会进入iter.Next()获取下一个ip，直到出现这个判断场景
```
if i.cur.Equal(r.RangeEnd) {
    i.rangeIdx += 1
    i.rangeIdx %= len(*i.rangeset)
    r = (*i.rangeset)[i.rangeIdx]

    i.cur = r.RangeStart
} else {
    #根据ip累增，获取下一个ip
    i.cur = ip.NextIP(i.cur)
}

if i.startIP == nil {
    i.startIP = i.cur
} else if i.rangeIdx == i.startRange && i.cur.Equal(i.startIP) {
    // IF we've looped back to where we started, give up
    return nil, nil
}
```
rangeIdx 是在多个range 场景，也就是多个ip域，当在其中一个ip域，选择到最后一个rangeEnd ip ，则进入下一个ip域，直到回到最初last_reserved_ip，则代表ip资源耗尽

#### cmdDel
##### 配置初始化（LoadIPAMConfig） 
##### 找寻IP文件，删除它
```
// N.B. This function eats errors to be tolerant and
// release as much as possible
func (s *Store) ReleaseByID(id string, ifname string) error {
    found := false
    match := strings.TrimSpace(id) + LineBreak + ifname
    found, err := s.ReleaseByKey(id, ifname, match)

    // For backwards compatibility, look for files written by a previous version
    if !found && err == nil {
        match := strings.TrimSpace(id)
    found, err = s.ReleaseByKey(id, ifname, match)
    }
    return err
}
```

