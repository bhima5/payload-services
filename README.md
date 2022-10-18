# payload-services


--controller code

 @Autowired
    public DailyAnalysisController(DailyAnalysisServiceImpl dasl) {
        this.dasl = dasl;
    }

    @CrossOrigin(origins = "*", allowedHeaders = "*")
    @PostMapping(value = "/getDailyAnalysis", consumes = {MediaType.APPLICATION_JSON_VALUE})
    public Mono<CXPResponse<HelperToConvertToJson>> getDailyAnalysis(@RequestBody DailyAnalysisInput dailyAnalysisInput) throws Exception {
        return dailyAnalysisHelper.getData(dailyAnalysisInput).onErrorResume(ex -> {
            CXPResponse<HelperToConvertToJson> res = new CXPResponse<>();
            return Mono.just(res);
        });
    }
    
  ----------helper code
  package onevz.cxp.payload.service.helperofdailyanalysis;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonMappingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import onevz.cxp.core.model.CXPResponse;
import onevz.cxp.payload.model.DailyAnalysisReport.*;
import onevz.cxp.payload.service.DailyAnalysisServiceImpl;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

import java.util.*;
import java.util.stream.Collectors;

@Component
public class DailyAnalysisHelper {

    @Autowired
    DailyAnalysisServiceImpl dasl;

    ObjectMapper mapper = new ObjectMapper();

    public Mono<CXPResponse<HelperToConvertToJson>> getData(DailyAnalysisInput dailyAnalysisInput) throws Exception {
        HelperToConvertToJson json = new HelperToConvertToJson();
        ArrayList<OrderFlow> orderFlow = dailyAnalysisInput.getOrderFlow();
        return dasl.getOrder(dailyAnalysisInput.getOrderFlow()).map(x -> {

            List<String> a = x.get("apiCompare");

            List<String> b = x.get("dataCompare");

            List<String> c = x.get("dataSync");


            List<ApiCompare> api = changeType(a);
            List<DataCompare> data = changeType1(b);
            List<DataSync> sync = changeType2(c);
            json.setDataCompare(data);
            json.setDataSync(sync);
            json.setApiCompare(api);

            List<DataSyncCount> ds = DailyAnalysisHelper.fetchDataSync(sync);
            List<DataCompareCount> dc = DailyAnalysisHelper.fetchDataCompare(data);
            List<ApiCompareCount> ac = DailyAnalysisHelper.fetchApiCompare(api);
            List<OrderAndType> oat = new ArrayList<>();
            for (OrderFlow of : orderFlow) {
                OrderAndType o = new OrderAndType();
                o.setType(of.getType());
                o.setOrdersCount(of.getOrderlist().size());
                oat.add(o);
            }
            json.setSummary(new Summary(oat, ds, dc, ac));
            CXPResponse<HelperToConvertToJson> res = new CXPResponse<>();
            res.setData(json);
            return res;
        });
    }

    private List<DataSync> changeType2(List<String> c) {
        ArrayList<DataSync> convertValues = (ArrayList<DataSync>) c.stream().map(itm -> {
            DataSync data = null;
            try {
                data = mapper.readValue(itm, DataSync.class);
            } catch (JsonMappingException e) {
                throw new RuntimeException(e);
            } catch (JsonProcessingException e) {
                throw new RuntimeException(e);
            }
            return data;
        }).collect(Collectors.toList());
        return convertValues;

    }

    private List<DataCompare> changeType1(List<String> b) {
        ArrayList<DataCompare> convertValues = (ArrayList<DataCompare>) b.stream().map(itm -> {
            DataCompare data = null;
            try {
                data = mapper.readValue(itm, DataCompare.class);
            } catch (JsonMappingException e) {
                throw new RuntimeException(e);
            } catch (JsonProcessingException e) {
                throw new RuntimeException(e);
            }
            return data;
        }).collect(Collectors.toList());
        return convertValues;

    }

    public List<ApiCompare> changeType(List<String> dummy) {
        ArrayList<ApiCompare> convertValues = (ArrayList<ApiCompare>) dummy.stream().map(itm -> {
            ApiCompare data = null;
            try {
                data = mapper.readValue(itm, ApiCompare.class);
            } catch (JsonMappingException e) {
                throw new RuntimeException(e);
            } catch (JsonProcessingException e) {
                throw new RuntimeException(e);
            }
            return data;
        }).collect(Collectors.toList());
        return convertValues;
    }

    public static List<DataSyncCount> fetchDataSync(List<DataSync> var1) {
        List<DataSyncCount> datasync = new ArrayList<DataSyncCount>();
        Map<String, DataSyncCount> ds = new HashMap<>();
        for (DataSync data : var1) {
            if (ds.containsKey(data.getFlow_Type())) {
                DataSyncCount var = ds.get(data.getFlow_Type());
                var.setSuccess(var.getSuccess() + data.getSuccess());
                var.setFailure(var.getFailure() + data.getFailure());
                var.setIn_processing(var.getIn_processing() + data.getInProgress());
                ds.replace(data.getFlow_Type(), var);
            } else {
                ds.put(data.getFlow_Type(), new DataSyncCount(data.getSuccess(), data.getFailure(), data.getInProgress()));
            }
        }
        for (String key : ds.keySet()) {
            DataSyncCount data = ds.get(key);
            data.setType(key);
            datasync.add(ds.get(key));
        }
        return datasync;
    }

    public static List<DataCompareCount> fetchDataCompare(List<DataCompare> var2) {
        List<DataCompareCount> datacompare = new ArrayList<DataCompareCount>();
        Map<String, DataCompareCount> dc = new HashMap<>();
        for (DataCompare data : var2) {
            if (dc.containsKey(data.getFlow_Type())) {
                DataCompareCount var = dc.get(data.getFlow_Type());
                var.setSuccess(var.getSuccess() + data.getSuccess());
                var.setFailure(var.getFailure() + data.getFailure());
                dc.replace(data.getFlow_Type(), var);
            } else {
                dc.put(data.getFlow_Type(), new DataCompareCount(data.getSuccess(), data.getFailure()));
            }
        }
        for (Map.Entry<String, DataCompareCount> com : dc.entrySet()) {
            DataCompareCount data = com.getValue();
            data.setType(com.getKey());
            datacompare.add(data);
        }
        return datacompare;
    }

    public static List<ApiCompareCount> fetchApiCompare(List<ApiCompare> var3) {
        List<ApiCompareCount> apicompare = new ArrayList<>();
        Map<String, ApiCompareCount> ac = new HashMap<>();
        for (ApiCompare data : var3) {
            if (ac.containsKey(data.getFlow_Type())) {
                ApiCompareCount var = ac.get(data.getFlow_Type());
                var.setSuccess(var.getSuccess() + data.getSuccess());
                var.setFailure(var.getFailure() + data.getFailure());
                ac.replace(data.getFlow_Type(), var);
            } else {
                ac.put(data.getFlow_Type(), new ApiCompareCount(data.getSuccess(), data.getFailure()));
            }
        }
        Iterator<Map.Entry<String, ApiCompareCount>> itr = ac.entrySet().iterator();
        while (itr.hasNext()) {
            Map.Entry<String, ApiCompareCount> pair = itr.next();
            ApiCompareCount compare = pair.getValue();
            compare.setType(pair.getKey());
            apicompare.add(compare);
        }
        return apicompare;
    }

}


-----service code

 private DailyAnalysisHelper dailyAnalysisHelper;

    ObjectMapper mapper = new ObjectMapper();

    public Mono<Map<String, List<String>>> getOrder(List<OrderFlow> listOfFlow) throws JsonProcessingException, Exception {
        Map<String, List<String>> resultMap = new HashMap<>();

        Flux<Map<String, Mono<List<String>>>> futureRequests =
//                Flux.fromIterable(listOfFLow).flatMap(x->)
                Flux.fromIterable(listOfFlow.parallelStream().filter(x -> x != null)
                        .map(x -> {
                            String typeId = x.getType();
//                            Map<String, Mono<List<String>>> resultMap1 = new HashMap<>();
                            try {
                                return dao.getResults1(typeId, x.getOrderlist(), x.getLocationlist());  //x.getLocationlist(), x.getOrderlist()

                            } catch (JsonProcessingException e) {
                                throw new RuntimeException(e);
                            }
//                            return resultMap1;
                        }).collect(Collectors.toList()));

//        Flux.fromIterable(List.of(futureRequests));

        futureRequests.doOnNext(s -> s.entrySet().stream().forEach(x -> {
            List<String> list = new ArrayList<String>();
            x.getValue().doOnNext(y->list.addAll(y)).subscribe();
            System.out.println("his "+x.getKey());
            System.out.println("his "+list);
            resultMap.computeIfAbsent(x.getKey(), k -> new ArrayList<>()).addAll(list);
        })).subscribe();
        return Mono.just(resultMap).delayElement(Duration.ofNanos(20));
    }

    public List<String> helpe(List<String> a){

        return a;
    }


}


-----
public class HelperToConvertToJson {

    Summary summary;
    List<DataSync> dataSync;
    List<DataCompare> dataCompare;
    List<ApiCompare> apiCompare;
}

------------
@Data
public class ApiCompare {

    @JsonProperty(value = "Flow_Type")
    private String Flow_Type;
    @JsonProperty(value = "api_type")
    private String Api_Type;
    @JsonProperty(value = "success")
    private int Success;
    @JsonProperty(value = "failed")
    private int Failure;
}

-----------------------
@Data
public class DataCompare {

    @JsonProperty(value = "Flow_Type")
    private String Flow_Type;
    @JsonProperty(value="milestones")
    private String Milestone;
    @JsonProperty(value="success")
    private int Success;
    @JsonProperty(value="failure")
    private int Failure;
}

-------
@Data
@Component
public class DataSync {

    @JsonProperty(value = "Flow_Type")
    private String Flow_Type;
    @JsonProperty(value = "type_def_name")
    public String Type_Def_Name;
    @JsonProperty(value = "success")
    public int Success;
    @JsonProperty(value = "failure")
    public int Failure;
    @JsonProperty(value = "in_processing")
    public int InProgress;
}
-------------
@Data
public class DataSyncCount {
    int success;
    int failure;
    int in_processing;
    String Type;
    public DataSyncCount(int success, int failure, int inProgress) {
        this.success=success;
        this.failure=failure;
        this.in_processing=inProgress;
    }
    public DataSyncCount(int success, int failure, int inProgress,String type) {
        this.success=success;
        this.failure=failure;
        this.in_processing=inProgress;
        this.Type=type;
    }
}
---------------------

@Data
public class DataCompareCount {
    int success;
    int failure;

    String type;
    public DataCompareCount(int success, int failure) {
        this.success=success;
        this.failure=failure;
    }
    public DataCompareCount(int success, int failure,String type) {
        this.success=success;
        this.failure=failure;
        this.type=type;
    }
}
---------------

@Data
public class ApiCompareCount {
    int success;
    int failure;

    String type;
    public ApiCompareCount(int success, int failure) {
        this.success=success;
        this.failure=failure;
    }

    public ApiCompareCount(int success, int failure,String type) {
        this.success=success;
        this.failure=failure;
        this.type=type;
    }
}
--------------------------
@Data
public class OrderAndType {

    private int ordersCount;
    private String type;
}
--------------------
@Data
@Component
public class DailyAnalysisInput {

    @JsonProperty(value="orderFlow")
    public ArrayList<OrderFlow> orderFlow;

}
------------------
@JsonInclude(Include.NON_NULL)
public class CXPResponse<D> implements Serializable {
    private static final long serialVersionUID = 1L;
    private Meta meta;
    @JsonInclude(Include.NON_EMPTY)
    private List<CXPError> errors;
    private D data;

    public CXPResponse() {
    }

    public Meta getMeta() {
        return this.meta;
    }

    public void setMeta(Meta meta) {
        this.meta = meta;
    }

    public D getData() {
        return this.data;
    }

    public void setData(D data) {
        this.data = data;
    }

    public List<CXPError> getErrors() {
        if (this.errors == null) {
            this.errors = new ArrayList();
        }

        return this.errors;
    }

    public void addError(CXPError error) {
        this.getErrors().add(error);
    }
}
-----------------
