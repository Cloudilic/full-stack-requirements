## Main requirements
- JavaScript ES6+
- TypeScript [Quick Reference](https://www.youtube.com/playlist?list=PLDoPjvoNmBAy532K9M_fjiAmrJ0gkCyLJ) (Classes do not matter at this time)<br>

## Frontend
- React.js (Hooks)
- Next.js (Choose any reference that suits you)
- Redux Toolkit [Quick Reference](https://redux-toolkit.js.org/introduction/getting-started)

## Backend
- Node.js
- Express.js
- MongoDB (Mongoose.js ODM)

## Libraries/tools (main)
- ReactFlow [Quick Reference](https://reactflow.dev/api-reference)
- Styled-Components / SCSS 
- Axios [Quick Reference](https://www.npmjs.com/package/axios)

## Deploy
- AWS ES2 (Linux Server)
- Remote Desktop Connection (Xrdp Server)

## Example of OpenaAI block (Frontend (Python API))
```js
// POSTMAN
// https://python.dragify.ai/api/llm/openai/query_openai
{
  "query": "Extract the Annotated data in a json format.",
  "model": "gpt-4o",
  "instructions": "You are an AI assistant capable of analyzing images. Provide detailed descriptions of images when asked.",
  "image_url": { // optional
    "url": "https://dragify-file-uploader.s3.us-east-1.amazonaws.com/1725577705178_annotated_image.png"
  },
  "history": [], // optional
  "relevant_text": null // optional
}
```
```tsx
import React from 'react';
import { NodeProps, useEdges, useNodes, useReactFlow, Node, Edge } from 'reactflow';
import axios from 'axios';
import { Select } from '../../../form/select';
import { Textarea } from '../../../form/textarea';
import { operationType, selectDataType } from '../../../../services/types';
import { Main } from './style';
import { addUpdateNodeToQueue } from '../../../../services/method';

// ---
export function OpenAI({ id, data }: NodeProps<operationType>) {
   const [models, setModels] = React.useState<selectDataType[]>([
      { id: 1, name: 'gpt-4o-mini', isActive: true }, { id: 2, name: 'gpt-4o', isActive: false },
   ]);
   const { setNodes } = useReactFlow();
   const nodes = useNodes();
   const edges = useEdges();

   // ---
   React.useEffect(() => {
      if (!data.value) {
         setNodes((nds) => nds.map((node) => {
            if (node.id === id) return { ...node, data: { ...node.data, value: { model: models[0].name, instructions: 'You are a helpful Assistant.' }}};
            else return node;
         }));
      } else {
         setModels(models => models.map(model => ({ ...model, isActive: model.name === data.value.model })));
      }
   // eslint-disable-next-line react-hooks/exhaustive-deps
   }, []);

   // ---
   const handleSelectModel = (selectId: number) => {
      setModels((models) => models.map((model) => {
         if (model.id === selectId) return { ...model, isActive: true };
         else return { ...model, isActive: false };
      }));
      setNodes((nds) => nds.map((node) => {
         if (node.id === id) return { ...node, data: { ...node.data, value: { ...node.data.value, model: models.find(m => m.id === selectId)?.name }}};
         else return node;
      }));
   }
   // ---
   const handleChangeInstructions = React.useCallback((e: React.ChangeEvent<HTMLTextAreaElement>) => {
      setNodes((nds) => nds.map((node) => {
         if (node.id === id) return { ...node, data: { ...node.data, value: { ...node.data.value, instructions: e.target.value }}};
         else return node;
      }));
   // eslint-disable-next-line react-hooks/exhaustive-deps
   }, []);
   
   // @ts-ignore
   const targetInput = edges.filter(edge => edge.target === id && nodes.some(node => node.id === edge.source && !external.includes(node.data?.action)));
   const relationsInput = nodes.filter(node => targetInput.some(edge => edge.source === node.id));
   // @ts-ignore
   const targetExternal = edges.filter(edge => edge.target === id && nodes.some(node => node.id === edge.source && external.includes(node.data?.action)));
   const relationsExternal = nodes.filter(node => targetExternal.some(edge => edge.source === node.id));
   
   return (
      <Main>
         <Select title="Model" data={ models } handleSelectItem={ handleSelectModel } />
         <Textarea title="Instructions" value={ data.value?.instructions } handleChange={ handleChangeInstructions } />
         <div className="title">Inputs</div>
         <div className="inputs nodrag nowheel">
            <div className="item">
               <div>
                  <span className={ relationsInput.length ? 'green' : 'red' }></span>
                  <span>User Input</span> 
               </div>
               <ul>{ // @ts-ignore
                  relationsInput.length ? relationsInput.map(node => (<li key={ node.id }>{ node.data?.index }</li>)) : <li>N/A</li>
               }</ul>
            </div>
            <div className="item">
               <div>
                  <span className={ relationsExternal.length ? 'green' : 'red' }></span>
                  <span>External Knowledge</span>
               </div>
               <ul>{ // @ts-ignore
                  relationsExternal.length ? relationsExternal.map(node => (<li key={ node.id }>{ node.data?.index }</li>)) : <li>N/A</li>
               }</ul>
            </div>
         </div>
      </Main>
   );
}

// ---
const external = ['docSearch', 'urlSearch', 'awsS3', 'azureStorage', 'googleDrive', 'oneDrive'];

// ---
export const handleOpenAIRun = async ({ id, nodes, setNodes, edges, isNeedValue } : {
   id: string, nodes: Node<any>[], setNodes: (payload: Node<any>[] | ((nodes: Node<any>[]) => Node<any>[])) => void, 
   edges: Edge<any>[], isNeedValue?: boolean
}) => {
   const item = nodes.find(node => node.id === id) as Node<operationType>;
   const targets: operationType[] = [], sources: operationType[] = [];

   for (const edge of edges) {
      if (edge.target === item.id) {
         for (const node of nodes) { if (node.id === edge.source) targets.push({ id: node.id, ...node.data })};

      } else if (edge.source === item.id) {
         for (const node of nodes) {
            if (node.id === edge.target) {
               if (node.data.outputType && !isNeedValue) return ''; 
               else sources.push({ id: node.id, ...node.data });
            }
         }
      }
   }
   if (!sources.length) return '';
   const request = { 
      model: item.data.value.model, 
      query: '', 
      instructions: item.data.value.instructions,
      relevant_text: ''
   };

   for (let item of targets) {         
      if (item.action === 'input' && item.value) request.query += `${item.value}\n`;
      else if (item.executeRun) {
         if (item.outputTypeSub === 'fileURL') {
            let file = await item.executeRun({ id: item.id as string, nodes, setNodes, edges, isNeedValue: true });
            if (file) {
               file = JSON.parse(file); // @ts-ignore
               if (file.type.startsWith('image')) request.image_url = { url: file.url };
            }
         } else if (item.outputType === 'text') {
            if (item.action === 'docSearch') request.relevant_text += (await item.executeRun({ id: item.id as string, nodes, setNodes, edges, isNeedValue: true }));
            else request.query += (await item.executeRun({ id: item.id as string, nodes, setNodes, edges, isNeedValue: true }));

         }
      }
   }
   if (request.query) {
      const response = (await axios.post('/llm/openai/query_openai', request)).data;

      addUpdateNodeToQueue(setNodes, (nodes) => nodes.map((node) => {
         if (sources.some(source => source.id === node.id && source.action === 'output')) return { ...node, data: { ...node.data, value: response.answer }};
         else return node;
      }));
      if (isNeedValue) return response.answer;
   } else return '';
}
```
```tsx
import styled from 'styled-components';

export const Main = styled.div`
   min-width: 335px;
   padding: 7.5px;

   .textarea {
      margin: 35px 0 0;
   }
   > .title {
      position: relative;
      top: calc(25px / 2);
      left: 7.5px;
      display: inline-flex;
      align-items: center;
      background-color: #F6F6F6;
      border: 1px solid rgb(220, 220, 220);
      padding: 0 7.5px;
      height: 25px;
      border-radius: 5px;
      z-index: 99;
   }
   .inputs {
      display: flex;
      flex-direction: column;
      justify-content: center;
      background-color: #F6F6F6;
      border: 1px solid rgb(220, 220, 220);
      border-radius: 5px;
      padding: 25px 15px 15px;
      overflow: auto;
      cursor: default;

      .item {
         display: flex;
         font-size: 15px;

         &:not(:last-of-type) { margin-bottom: 10px; }
         > div {
            display: flex;

            span:first-of-type {
               width: 10px;
               height: 10px;
               background-color: red;
               margin-right: 7.5px;
               border-radius: 2px;

               &.red {
                  background-color: red;
                  box-shadow: 0 0 3px red;
               }
               &.green {
                  background-color: #339966;
                  box-shadow: 0 0 3px #339966;
               }
            }
            span:last-of-type {
               width: 147px;
               opacity: 0.85;
            }
         }
         ul li { opacity: 0.85; }
      }
   }
`;
```
